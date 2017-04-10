

# TextView源码解析

***
## 1.简介
***
> TextView作为Android系统上显示和排版文字以及提供对文字的增删改查、图文混排等功能的控件，内部是相对比较复杂的。这么一个复杂的控件自然需要依赖于一些其他的辅助类，例如：Layout以及Layout的相关子类、Span相关的类、MovementMethod接口、TransformationMethod接口等。这篇文章主要介绍TextView的结构和内部处理文字的流程以及TextView相关的辅助类在TextView处理文字过程中的作用。

## 2.TextView的内部结构和辅助类
***
TextView内部除了继承自View的相关属性和measure、layout、draw步骤，还包括：

 1. **Layout**: TextView的文字排版、折行策略以及文本绘制都是在Layout里面完成的，TextView的自身测量也受Layout的影响。Layout是TextView执行setText方法后，由TextView内部创建的实例，并不能由外部提供。可以用getLayout()方法获取。
 2. **TransformationMethod**: 用来处理最终的显示结果的类，例如显示密码的时候把密码转换成圆点。这个类并不直接影响TextView内部储存的Text，只影响显示的结果。
 3. **MovementMethod**: 用来处理TextView内部事件响应的类，可以针对TextView内文本的某一个区域做软键盘输入或者触摸事件的响应。
 4. **Drawables**: TextView的静态内部类，用来处理和储存TextView的CompoundDrawables,包括TextView的上下左右的Drawable以及错误提示的Drawable。
 5. **Spans**: Spans并不是特定的某一个类或者实现了某一个接口的类。它可以是任意类型，Spans实际上做的事情是在TextView的内部的text的某一个区域做标记。其中有部分Spans可以影响TextView的绘制和测量，如ImageSpan、BackgroundColorSpan、AbsoluteSizeSpan。还有可以响应点击事件的ClickableSpan。
 6. **Editor**: TextView作为可编辑文本控件的时候(EditText)，使用Editor来处理文本的区域选择处理和判断、拼写检查、弹出文本菜单等。
 7. **InputConnection**: EditText的文本输入部分是在TextView中完成的。而InputConnection是软键盘和TextView之间的桥梁，所有的软键盘的输入文字、修改文字和删除文字都是通过InputConnection传递给TextView的。

## 3.TextView的onTouchEvent处理

***

TextView内部能处理触摸事件的，包括自身的触摸处理、Editor的onTouchEvent、MovementMethod的onTouchEvent。Editor的onTouchEvent主要处理出于编辑状态下的触摸事件，比如点击选中、长按等。MovementMethod则主要负责文本内部有Span的时候的相关处理，比较常见的就是LinkMovementMethod处理ClickableSpan的点击事件。我们来看一下TextView内部对这些触摸事件的处理和优先级的分配：

```java
public boolean onTouchEvent(MotionEvent event) {
        final int action = event.getActionMasked();

        //当Editor不为空的时候，给Editor的双击事件预设值
        if (mEditor != null && action == MotionEvent.ACTION_DOWN) {
            if (mFirstTouch && (SystemClock.uptimeMillis() - mLastTouchUpTime) <=
                    ViewConfiguration.getDoubleTapTimeout()) {
                mEditor.mDoubleTap = true;
                mFirstTouch = false;
            } else {
                mEditor.mDoubleTap = false;
                mFirstTouch = true;
            }
        }

        if (action == MotionEvent.ACTION_UP) {
            mLastTouchUpTime = SystemClock.uptimeMillis();
        }

        //当Editor不为空，优先处理Editor的触摸事件
        if (mEditor != null) {
            mEditor.onTouchEvent(event);

            //由于Editor内部onTouchEvent实际上交给了mSelectionModifierCursorController处理，所以这边判断mSelectionModifierCursorController是否需要处理接下来的一系列事件，如果是则直接返回跳过下面的步骤
            if (mEditor.mSelectionModifierCursorController != null &&
                    mEditor.mSelectionModifierCursorController.isDragAcceleratorActive()) {
                return true;
            }
        }

        final boolean superResult = super.onTouchEvent(event);

        //处理API 23新加入的InsertionActinoMode
        if (mEditor != null && mEditor.mDiscardNextActionUp && action == MotionEvent.ACTION_UP) {
            mEditor.mDiscardNextActionUp = false;

            if (mEditor.mIsInsertionActionModeStartPending) {
                mEditor.startInsertionActionMode();
                mEditor.mIsInsertionActionModeStartPending = false;
            }
            return superResult;
        }

        final boolean touchIsFinished = (action == MotionEvent.ACTION_UP) &&
                (mEditor == null || !mEditor.mIgnoreActionUpEvent) && isFocused();

         if ((mMovement != null || onCheckIsTextEditor()) && isEnabled()
                && mText instanceof Spannable && mLayout != null) {
            boolean handled = false;

            //MovementMethod的触摸时间处理，如果MovementMethod类型是LinkMovementMethod则会处理文本内的所有ClickableSpan的点击
            if (mMovement != null) {
                handled |= mMovement.onTouchEvent(this, (Spannable) mText, event);
            }

            final boolean textIsSelectable = isTextSelectable();
            if (touchIsFinished && mLinksClickable && mAutoLinkMask != 0 && textIsSelectable) {
                
                //在文本可选择的情况下，默认是没有LinkMovementMethod来处理ClickableSpan相关的点击的，所以在文本可选择情况，TextView对所有的ClickableSpan进行统一处理
                ClickableSpan[] links = ((Spannable) mText).getSpans(getSelectionStart(),
                        getSelectionEnd(), ClickableSpan.class);

                if (links.length > 0) {
                    links[0].onClick(this);
                    handled = true;
                }
            }

            if (touchIsFinished && (isTextEditable() || textIsSelectable)) {
                final InputMethodManager imm = InputMethodManager.peekInstance();
                viewClicked(imm);
                if (!textIsSelectable && mEditor.mShowSoftInputOnFocus) {
                    handled |= imm != null && imm.showSoftInput(this, 0);
                }

                mEditor.onTouchUpEvent(event);

                handled = true;
            }

            if (handled) {
                return true;
            }
        }

        return superResult;
    }
```

## 4.TextView的创建Layout的过程

***

TextView内部并不仅仅只有一个用来显示文本内容的Layout，在设置了hint的时候，还需要有一个mHintLayout来处理hint的内容。如果设置了Ellipsize类型为Marquee时，还会有一个mSavedMarqueeModeLayout专门用来显示marquee效果。这些Layout都是通过内部的makeNewLayout方法来创建的：

```java
protected void makeNewLayout(int wantWidth, int hintWidth
                                 BoringLayout.Metrics boring,
                                 BoringLayout.Metrics hintBoring,
                                 int ellipsisWidth, boolean bringIntoView) {
        //如果当前有marquee动画，则先停止动画
        stopMarquee();

        mOldMaximum = mMaximum;
        mOldMaxMode = mMaxMode;

        mHighlightPathBogus = true;

        if (wantWidth < 0) {
            wantWidth = 0;
        }
        if (hintWidth < 0) {
            hintWidth = 0;
        }

        //文本对齐方式
        Layout.Alignment alignment = getLayoutAlignment();
        final boolean testDirChange = mSingleLine && mLayout != null &&
            (alignment == Layout.Alignment.ALIGN_NORMAL ||
             alignment == Layout.Alignment.ALIGN_OPPOSITE);
        int oldDir = 0;
        if (testDirChange) oldDir = mLayout.getParagraphDirection(0);
  
        //检测是否设置了ellipsize
        boolean shouldEllipsize = mEllipsize != null && getKeyListener() == null;
        final boolean switchEllipsize = mEllipsize == TruncateAt.MARQUEE &&
                mMarqueeFadeMode != MARQUEE_FADE_NORMAL;
        TruncateAt effectiveEllipsize = mEllipsize;
        if (mEllipsize == TruncateAt.MARQUEE &&
                mMarqueeFadeMode == MARQUEE_FADE_SWITCH_SHOW_ELLIPSIS) {
            effectiveEllipsize = TruncateAt.END_SMALL;
        }
    
        //文本方向
        if (mTextDir == null) {
            mTextDir = getTextDirectionHeuristic();
        }

        //创建主Layout
        mLayout = makeSingleLayout(wantWidth, boring, ellipsisWidth, alignment, shouldEllipsize,
                effectiveEllipsize, effectiveEllipsize == mEllipsize);
  
        //非常规的Marquee模式下，需要创建mSavedMarqueeModeLayout来保存marquee动画时所用的Layout，并且在动画期间把它和TextView的主Layout对换
        if (switchEllipsize) {
            TruncateAt oppositeEllipsize = effectiveEllipsize == TruncateAt.MARQUEE ?
                    TruncateAt.END : TruncateAt.MARQUEE;
            mSavedMarqueeModeLayout = makeSingleLayout(wantWidth, boring, ellipsisWidth, alignment,
                    shouldEllipsize, oppositeEllipsize, effectiveEllipsize != mEllipsize);
        }

        shouldEllipsize = mEllipsize != null;
        mHintLayout = null;

        //判断是否需要创建hintLayout
        if (mHint != null) {
            if (shouldEllipsize) hintWidth = wantWidth;

            if (hintBoring == UNKNOWN_BORING) {
                hintBoring = BoringLayout.isBoring(mHint, mTextPaint, mTextDir,
                                                   mHintBoring);
                if (hintBoring != null) {
                    mHintBoring = hintBoring;
                }
            }

            //判断是否为boring，如果是则创建BoringLayout
            if (hintBoring != null) {
                if (hintBoring.width <= hintWidth &&
                    (!shouldEllipsize || hintBoring.width <= ellipsisWidth)) {
                    if (mSavedHintLayout != null) {
                        mHintLayout = mSavedHintLayout.
                                replaceOrMake(mHint, mTextPaint,
                                hintWidth, alignment, mSpacingMult, mSpacingAdd,
                                hintBoring, mIncludePad);
                    } else {
                        mHintLayout = BoringLayout.make(mHint, mTextPaint,
                                hintWidth, alignment, mSpacingMult, mSpacingAdd,
                                hintBoring, mIncludePad);
                    }

                    mSavedHintLayout = (BoringLayout) mHintLayout;
                } else if (shouldEllipsize && hintBoring.width <= hintWidth) {
                    if (mSavedHintLayout != null) {
                        mHintLayout = mSavedHintLayout.
                                replaceOrMake(mHint, mTextPaint,
                                hintWidth, alignment, mSpacingMult, mSpacingAdd,
                                hintBoring, mIncludePad, mEllipsize,
                                ellipsisWidth);
                    } else {
                        mHintLayout = BoringLayout.make(mHint, mTextPaint,
                                hintWidth, alignment, mSpacingMult, mSpacingAdd,
                                hintBoring, mIncludePad, mEllipsize,
                                ellipsisWidth);
                    }
                }
            }
          
            //不是boring的状态下，用StaticLayout来创建
            if (mHintLayout == null) {
                StaticLayout.Builder builder = StaticLayout.Builder.obtain(mHint, 0,
                        mHint.length(), mTextPaint, hintWidth)
                        .setAlignment(alignment)
                        .setTextDirection(mTextDir)
                        .setLineSpacing(mSpacingAdd, mSpacingMult)
                        .setIncludePad(mIncludePad)
                        .setBreakStrategy(mBreakStrategy)
                        .setHyphenationFrequency(mHyphenationFrequency);
                if (shouldEllipsize) {
                    builder.setEllipsize(mEllipsize)
                            .setEllipsizedWidth(ellipsisWidth)
                            .setMaxLines(mMaxMode == LINES ? mMaximum : Integer.MAX_VALUE);
                }
                mHintLayout = builder.build();
            }
        }

        if (bringIntoView || (testDirChange && oldDir != mLayout.getParagraphDirection(0))) {
            registerForPreDraw();
        }

        //判断是否需要开始Marquee动画
        if (mEllipsize == TextUtils.TruncateAt.MARQUEE) {
            if (!compressText(ellipsisWidth)) {
                final int height = mLayoutParams.height;
                if (height != LayoutParams.WRAP_CONTENT && height != LayoutParams.MATCH_PARENT) {
                    startMarquee();
                } else {
                    mRestartMarquee = true;
                }
            }
        }

        if (mEditor != null) mEditor.prepareCursorControllers();
    }
```

TextView的布局创建过程涉及到一个boring的概念，boring是指布局所用的文本里面不包含任何Span，所有的文本方向都是从左到右的布局，并且仅需一行就能显示完全的布局。这种情况下，TextView会使用BoringLayout类来创建相关的布局，以节省不必要的文本测量以及文本折行、Span宽度、文本方向等的计算。下面我们来看一下makeNewLayout中使用频率比较高的makeSingleLayout的代码:

```java
private Layout makeSingleLayout(int wantWidth, BoringLayout.Metrics boring, int ellipsisWidth,
            Layout.Alignment alignment, boolean shouldEllipsize, TruncateAt effectiveEllipsize,
            boolean useSaved) {
        Layout result = null;
        //判断是否Spannable，如果是则用DynamicLayout类来创建布局，DynamicLayout内部实际也是使用StaticLayout来做文本的测量绘制，并在StaticLayout的基础上增加了文本或者Span改变时的监听，及时对文本或者Span的变化做出反应。
        if (mText instanceof Spannable) {
            result = new DynamicLayout(mText, mTransformed, mTextPaint, wantWidth,
                    alignment, mTextDir, mSpacingMult, mSpacingAdd, mIncludePad,
                    mBreakStrategy, mHyphenationFrequency,
                    getKeyListener() == null ? effectiveEllipsize : null, ellipsisWidth);
        } else {
            //如果boring是未知状态，则重新判断一次是否boring
            if (boring == UNKNOWN_BORING) {
                boring = BoringLayout.isBoring(mTransformed, mTextPaint, mTextDir, mBoring);
                if (boring != null) {
                    mBoring = boring;
                }
            }

            //根据boring的属性来创建对应的布局，如果有mSavedLayout则从mSavedLayout创建
            if (boring != null) {
                if (boring.width <= wantWidth &&
                        (effectiveEllipsize == null || boring.width <= ellipsisWidth)) {
                    if (useSaved && mSavedLayout != null) {
                        //从之前保存的Layout中创建
                        result = mSavedLayout.replaceOrMake(mTransformed, mTextPaint,
                                wantWidth, alignment, mSpacingMult, mSpacingAdd,
                                boring, mIncludePad);
                    } else {
                        //创建新的Layout
                        result = BoringLayout.make(mTransformed, mTextPaint,
                                wantWidth, alignment, mSpacingMult, mSpacingAdd,
                                boring, mIncludePad);
                    }

                    if (useSaved) {
                        mSavedLayout = (BoringLayout) result;
                    }
                } else if (shouldEllipsize && boring.width <= wantWidth) {
                    if (useSaved && mSavedLayout != null) {
                        result = mSavedLayout.replaceOrMake(mTransformed, mTextPaint,
                                wantWidth, alignment, mSpacingMult, mSpacingAdd,
                                boring, mIncludePad, effectiveEllipsize,
                                ellipsisWidth);
                    } else {
                        result = BoringLayout.make(mTransformed, mTextPaint,
                                wantWidth, alignment, mSpacingMult, mSpacingAdd,
                                boring, mIncludePad, effectiveEllipsize,
                                ellipsisWidth);
                    }
                }
            }
        }
  
        //如果没有创建BoringLayout, 则使用StaticLayout类来创建布局
        if (result == null) {
            StaticLayout.Builder builder = StaticLayout.Builder.obtain(mTransformed,
                    0, mTransformed.length(), mTextPaint, wantWidth)
                    .setAlignment(alignment)
                    .setTextDirection(mTextDir)
                    .setLineSpacing(mSpacingAdd, mSpacingMult)
                    .setIncludePad(mIncludePad)
                    .setBreakStrategy(mBreakStrategy)
                    .setHyphenationFrequency(mHyphenationFrequency);
            if (shouldEllipsize) {
                builder.setEllipsize(effectiveEllipsize)
                        .setEllipsizedWidth(ellipsisWidth)
                        .setMaxLines(mMaxMode == LINES ? mMaximum : Integer.MAX_VALUE);
            }
            result = builder.build();
        }
        return result;
    }
```

## 5.TextView的文字处理和绘制

---

TextView主要的文字排版和渲染并不是在TextView里面完成的，而是由Layout类来处理文字排版工作。在单纯地使用TextView来展示静态文本的时候，这件事情则是由Layout的子类StaticLayout来完成的。

StaticLayout接收到字符串后，首先做的事情是根据字符串里面的换行符对字符串进行拆分。

```java
for (int paraStart = bufStart; paraStart <= bufEnd; paraStart = paraEnd) {
            paraEnd = TextUtils.indexOf(source, CHAR_NEW_LINE, paraStart, bufEnd);
            if (paraEnd < 0)
                paraEnd = bufEnd;
            else
                paraEnd++;
```

拆分后的段落(Paragraph)被分配给辅助类MeasuredText进行测量得到每个字符的宽度以及每个段落的FontMetric。并通过LineBreaker进行折行的判断

```java
//把段落载入到MeasuredText中，并分配对应的缓存空间
measured.setPara(source, paraStart, paraEnd, textDir, b);
            char[] chs = measured.mChars;
            float[] widths = measured.mWidths;
            byte[] chdirs = measured.mLevels;
            int dir = measured.mDir;
            boolean easy = measured.mEasy;

```

            //把相关属性传给JNI层的LineBreaker
            nSetupParagraph(b.mNativePtr, chs, paraEnd - paraStart,
                    firstWidth, firstWidthLineCount, restWidth,
                    variableTabStops, TAB_INCREMENT, b.mBreakStrategy, b.mHyphenationFrequency);
```

            int fmCacheCount = 0;
            int spanEndCacheCount = 0;
            for (int spanStart = paraStart, spanEnd; spanStart < paraEnd; spanStart = spanEnd) {
                if (fmCacheCount * 4 >= fmCache.length) {
                    int[] grow = new int[fmCacheCount * 4 * 2];
                    System.arraycopy(fmCache, 0, grow, 0, fmCacheCount * 4);
                    fmCache = grow;
                }

                if (spanEndCacheCount >= spanEndCache.length) {
                    int[] grow = new int[spanEndCacheCount * 2];
                    System.arraycopy(spanEndCache, 0, grow, 0, spanEndCacheCount);
                    spanEndCache = grow;
                }

                if (spanned == null) {
                    spanEnd = paraEnd;
                    int spanLen = spanEnd - spanStart;
                    //段落没有Span的情况下，把整个段落交给MeasuredText计算每个字符的宽度和FontMetric
                    measured.addStyleRun(paint, spanLen, fm);
                } else {
                    spanEnd = spanned.nextSpanTransition(spanStart, paraEnd,
                            MetricAffectingSpan.class);
                    int spanLen = spanEnd - spanStart;
                    MetricAffectingSpan[] spans =
                            spanned.getSpans(spanStart, spanEnd, MetricAffectingSpan.class);
                    spans = TextUtils.removeEmptySpans(spans, spanned, MetricAffectingSpan.class);
                    //把对排版有影响的Span交给MeasuredText测量宽度并计算FontMetric
                    measured.addStyleRun(paint, spans, spanLen, fm);
                }

                //把测量后的FontMetric缓存下来方便后面使用
                fmCache[fmCacheCount * 4 + 0] = fm.top;
                fmCache[fmCacheCount * 4 + 1] = fm.bottom;
                fmCache[fmCacheCount * 4 + 2] = fm.ascent;
                fmCache[fmCacheCount * 4 + 3] = fm.descent;
                fmCacheCount++;

                spanEndCache[spanEndCacheCount] = spanEnd;
                spanEndCacheCount++;
            }

            nGetWidths(b.mNativePtr, widths);
            //计算段落中需要折行的位置，并返回折行的数量
            int breakCount = nComputeLineBreaks(b.mNativePtr, lineBreaks, lineBreaks.breaks,
                    lineBreaks.widths, lineBreaks.flags, lineBreaks.breaks.length);
```

计算完每一行的测量相关信息、Span宽高以及折行位置，就可以开始按照最终的行数一行一行地保存下来，以供后面绘制和获取对应文本信息的时候使用。

```java
for (int spanStart = paraStart, spanEnd; spanStart < paraEnd; spanStart = spanEnd) {
                spanEnd = spanEndCache[spanEndCacheIndex++];

                // 获取之前缓存的FontMetric信息
                fm.top = fmCache[fmCacheIndex * 4 + 0];
                fm.bottom = fmCache[fmCacheIndex * 4 + 1];
                fm.ascent = fmCache[fmCacheIndex * 4 + 2];
                fm.descent = fmCache[fmCacheIndex * 4 + 3];
                fmCacheIndex++;

                if (fm.top < fmTop) {
                    fmTop = fm.top;
                }
                if (fm.ascent < fmAscent) {
                    fmAscent = fm.ascent;
                }
                if (fm.descent > fmDescent) {
                    fmDescent = fm.descent;
                }
                if (fm.bottom > fmBottom) {
                    fmBottom = fm.bottom;
                }

                while (breakIndex < breakCount && paraStart + breaks[breakIndex] < spanStart) {
                    breakIndex++;
                }

                while (breakIndex < breakCount && paraStart + breaks[breakIndex] <= spanEnd) {
                    int endPos = paraStart + breaks[breakIndex];

                    boolean moreChars = (endPos < bufEnd);

                    //逐行把相关信息储存下来
                    v = out(source, here, endPos,
                            fmAscent, fmDescent, fmTop, fmBottom,
                            v, spacingmult, spacingadd, chooseHt, chooseHtv, fm, flags[breakIndex],
                            needMultiply, chdirs, dir, easy, bufEnd, includepad, trackpad,
                            chs, widths, paraStart, ellipsize, ellipsizedWidth,
                            lineWidths[breakIndex], paint, moreChars);

                    if (endPos < spanEnd) {
                        fmTop = fm.top;
                        fmBottom = fm.bottom;
                        fmAscent = fm.ascent;
                        fmDescent = fm.descent;
                    } else {
                        fmTop = fmBottom = fmAscent = fmDescent = 0;
                    }

                    here = endPos;
                    breakIndex++;

                    if (mLineCount >= mMaximumVisibleLineCount) {
                        return;
                    }
                }
            }
```

这样StaticLayout的排版过程就完成了。文本的绘制则是交给父类Layout来做的，Layout的绘制分为两大部分，drawBackground和drawText。drawBackground做的事情是如果文本内有LineBackgroundSpan则绘制所有的LineBackgroundSpan，然后判断是否有高亮背景(文本选中的背景)，如果有则绘制高亮背景。

```java
public void drawBackground(Canvas canvas, Path highlight, Paint highlightPaint,
            int cursorOffsetVertical, int firstLine, int lastLine) {
  
        //判断并绘制LineBackgroundSpan
        if (mSpannedText) {
            if (mLineBackgroundSpans == null) {
                mLineBackgroundSpans = new SpanSet<LineBackgroundSpan>(LineBackgroundSpan.class);
            }

            Spanned buffer = (Spanned) mText;
            int textLength = buffer.length();
            mLineBackgroundSpans.init(buffer, 0, textLength);

            if (mLineBackgroundSpans.numberOfSpans > 0) {
                int previousLineBottom = getLineTop(firstLine);
                int previousLineEnd = getLineStart(firstLine);
                ParagraphStyle[] spans = NO_PARA_SPANS;
                int spansLength = 0;
                TextPaint paint = mPaint;
                int spanEnd = 0;
                final int width = mWidth;
                //逐行绘制LineBackgroundSpan
                for (int i = firstLine; i <= lastLine; i++) {
                    int start = previousLineEnd;
                    int end = getLineStart(i + 1);
                    previousLineEnd = end;

                    int ltop = previousLineBottom;
                    int lbottom = getLineTop(i + 1);
                    previousLineBottom = lbottom;
                    int lbaseline = lbottom - getLineDescent(i);

                    if (start >= spanEnd) {
                        spanEnd = mLineBackgroundSpans.getNextTransition(start, textLength);
                        
                        spansLength = 0;
                        if (start != end || start == 0) {
                            //排除不在绘制范围内的LineBackgroundSpan
                            for (int j = 0; j < mLineBackgroundSpans.numberOfSpans; j++) {
                                if (mLineBackgroundSpans.spanStarts[j] >= end ||
                                        mLineBackgroundSpans.spanEnds[j] <= start) continue;
                                spans = GrowingArrayUtils.append(
                                        spans, spansLength, mLineBackgroundSpans.spans[j]);
                                spansLength++;
                            }
                        }
                    }
                    //对当前行内的LineBackgroundSpan进行绘制
                    for (int n = 0; n < spansLength; n++) {
                        LineBackgroundSpan lineBackgroundSpan = (LineBackgroundSpan) spans[n];
                        lineBackgroundSpan.drawBackground(canvas, paint, 0, width,
                                ltop, lbaseline, lbottom,
                                buffer, start, end, i);
                    }
                }
            }
            mLineBackgroundSpans.recycle();
        }

        //判断并绘制高亮背景(即选中的文本)
        if (highlight != null) {
            if (cursorOffsetVertical != 0) canvas.translate(0, cursorOffsetVertical);
            canvas.drawPath(highlight, highlightPaint);
            if (cursorOffsetVertical != 0) canvas.translate(0, -cursorOffsetVertical);
        }
    }
```



drawText用来逐行绘制Layout的文本、影响显示效果的Span、以及Emoji表情等。当有Emoji或者Span的时候，实际绘制工作交给TextLine类来完成。

```java
public void drawText(Canvas canvas, int firstLine, int lastLine) {
        int previousLineBottom = getLineTop(firstLine);
        int previousLineEnd = getLineStart(firstLine);
        ParagraphStyle[] spans = NO_PARA_SPANS;
        int spanEnd = 0;
        TextPaint paint = mPaint;
        CharSequence buf = mText;

        Alignment paraAlign = mAlignment;
        TabStops tabStops = null;
        boolean tabStopsIsInitialized = false;

        //获取TextLine实例
        TextLine tl = TextLine.obtain();

        //逐行绘制文本
        for (int lineNum = firstLine; lineNum <= lastLine; lineNum++) {
            int start = previousLineEnd;
            previousLineEnd = getLineStart(lineNum + 1);
            int end = getLineVisibleEnd(lineNum, start, previousLineEnd);

            int ltop = previousLineBottom;
            int lbottom = getLineTop(lineNum + 1);
            previousLineBottom = lbottom;
            int lbaseline = lbottom - getLineDescent(lineNum);

            int dir = getParagraphDirection(lineNum);
            int left = 0;
            int right = mWidth;

            if (mSpannedText) {
                Spanned sp = (Spanned) buf;
                int textLength = buf.length();
                //检测是否段落的第一行
                boolean isFirstParaLine = (start == 0 || buf.charAt(start - 1) == '\n');

                //获得所有的段落风格相关的Span
                if (start >= spanEnd && (lineNum == firstLine || isFirstParaLine)) {
                    spanEnd = sp.nextSpanTransition(start, textLength,
                                                    ParagraphStyle.class);
                    spans = getParagraphSpans(sp, start, spanEnd, ParagraphStyle.class);

                    paraAlign = mAlignment;
                    for (int n = spans.length - 1; n >= 0; n--) {
                        if (spans[n] instanceof AlignmentSpan) {
                            paraAlign = ((AlignmentSpan) spans[n]).getAlignment();
                            break;
                        }
                    }

                    tabStopsIsInitialized = false;
                }

                //获取影响行缩进的Span
                final int length = spans.length;
                boolean useFirstLineMargin = isFirstParaLine;
                for (int n = 0; n < length; n++) {
                    if (spans[n] instanceof LeadingMarginSpan2) {
                        int count = ((LeadingMarginSpan2) spans[n]).getLeadingMarginLineCount();
                        int startLine = getLineForOffset(sp.getSpanStart(spans[n]));
                        if (lineNum < startLine + count) {
                            useFirstLineMargin = true;
                            break;
                        }
                    }
                }
                for (int n = 0; n < length; n++) {
                    if (spans[n] instanceof LeadingMarginSpan) {
                        LeadingMarginSpan margin = (LeadingMarginSpan) spans[n];
                        if (dir == DIR_RIGHT_TO_LEFT) {
                            margin.drawLeadingMargin(canvas, paint, right, dir, ltop,
                                                     lbaseline, lbottom, buf,
                                                     start, end, isFirstParaLine, this);
                            right -= margin.getLeadingMargin(useFirstLineMargin);
                        } else {
                            margin.drawLeadingMargin(canvas, paint, left, dir, ltop,
                                                     lbaseline, lbottom, buf,
                                                     start, end, isFirstParaLine, this);
                            left += margin.getLeadingMargin(useFirstLineMargin);
                        }
                    }
                }
            }

            boolean hasTabOrEmoji = getLineContainsTab(lineNum);
            if (hasTabOrEmoji && !tabStopsIsInitialized) {
                if (tabStops == null) {
                    tabStops = new TabStops(TAB_INCREMENT, spans);
                } else {
                    tabStops.reset(TAB_INCREMENT, spans);
                }
                tabStopsIsInitialized = true;
            }

            //判断当前行的第五方式
            Alignment align = paraAlign;
            if (align == Alignment.ALIGN_LEFT) {
                align = (dir == DIR_LEFT_TO_RIGHT) ?
                    Alignment.ALIGN_NORMAL : Alignment.ALIGN_OPPOSITE;
            } else if (align == Alignment.ALIGN_RIGHT) {
                align = (dir == DIR_LEFT_TO_RIGHT) ?
                    Alignment.ALIGN_OPPOSITE : Alignment.ALIGN_NORMAL;
            }

            int x;
            if (align == Alignment.ALIGN_NORMAL) {
                if (dir == DIR_LEFT_TO_RIGHT) {
                    x = left + getIndentAdjust(lineNum, Alignment.ALIGN_LEFT);
                } else {
                    x = right + getIndentAdjust(lineNum, Alignment.ALIGN_RIGHT);
                }
            } else {
                int max = (int)getLineExtent(lineNum, tabStops, false);
                if (align == Alignment.ALIGN_OPPOSITE) {
                    if (dir == DIR_LEFT_TO_RIGHT) {
                        x = right - max + getIndentAdjust(lineNum, Alignment.ALIGN_RIGHT);
                    } else {
                        x = left - max + getIndentAdjust(lineNum, Alignment.ALIGN_LEFT);
                    }
                } else { // Alignment.ALIGN_CENTER
                    max = max & ~1;
                    x = ((right + left - max) >> 1) +
                            getIndentAdjust(lineNum, Alignment.ALIGN_CENTER);
                }
            }

            paint.setHyphenEdit(getHyphen(lineNum));
            Directions directions = getLineDirections(lineNum);
            if (directions == DIRS_ALL_LEFT_TO_RIGHT && !mSpannedText && !hasTabOrEmoji) {
                //没有任何Emoji或者span的时候，直接调用Canvas来绘制文本
                canvas.drawText(buf, start, end, x, lbaseline, paint);
            } else {
                //当有Emoji或者Span的时候，交给TextLine类来绘制
                tl.set(paint, buf, start, end, dir, directions, hasTabOrEmoji, tabStops);
                tl.draw(canvas, x, ltop, lbaseline, lbottom);
            }
            paint.setHyphenEdit(0);
        }

        TextLine.recycle(tl);
    }
```



我们下面再来看看TextLine是如何绘制有特殊情况的文本的

```java
void draw(Canvas c, float x, int top, int y, int bottom) {
        //判断是否有Tab或者Emoji
        if (!mHasTabs) {
            if (mDirections == Layout.DIRS_ALL_LEFT_TO_RIGHT) {
                drawRun(c, 0, mLen, false, x, top, y, bottom, false);
                return;
            }
            if (mDirections == Layout.DIRS_ALL_RIGHT_TO_LEFT) {
                drawRun(c, 0, mLen, true, x, top, y, bottom, false);
                return;
            }
        }

        float h = 0;
        int[] runs = mDirections.mDirections;
        RectF emojiRect = null;

        int lastRunIndex = runs.length - 2;
        //逐个绘制
        for (int i = 0; i < runs.length; i += 2) {
            int runStart = runs[i];
            int runLimit = runStart + (runs[i+1] & Layout.RUN_LENGTH_MASK);
            if (runLimit > mLen) {
                runLimit = mLen;
            }
            boolean runIsRtl = (runs[i+1] & Layout.RUN_RTL_FLAG) != 0;

            int segstart = runStart;
            for (int j = mHasTabs ? runStart : runLimit; j <= runLimit; j++) {
                int codept = 0;
                Bitmap bm = null;

                if (mHasTabs && j < runLimit) {
                    codept = mChars[j];
                    if (codept >= 0xd800 && codept < 0xdc00 && j + 1 < runLimit) {
                        codept = Character.codePointAt(mChars, j);
                        if (codept >= Layout.MIN_EMOJI && codept <= Layout.MAX_EMOJI) {
                            //获取Emoji对应的图像
                            bm = Layout.EMOJI_FACTORY.getBitmapFromAndroidPua(codept);
                        } else if (codept > 0xffff) {
                            ++j;
                            continue;
                        }
                    }
                }

                if (j == runLimit || codept == '\t' || bm != null) {
                    //绘制文字
                    h += drawRun(c, segstart, j, runIsRtl, x+h, top, y, bottom,
                            i != lastRunIndex || j != mLen);

                    if (codept == '\t') {
                        h = mDir * nextTab(h * mDir);
                    } else if (bm != null) {
                        float bmAscent = ascent(j);
                        float bitmapHeight = bm.getHeight();
                        float scale = -bmAscent / bitmapHeight;
                        float width = bm.getWidth() * scale;

                        if (emojiRect == null) {
                            emojiRect = new RectF();
                        }
                        //调整emoji图像绘制矩形
                        emojiRect.set(x + h, y + bmAscent,
                                x + h + width, y);
                        //绘制Emoji图像
                        c.drawBitmap(bm, null, emojiRect, mPaint);
                        h += width;
                        j++;
                    }
                    segstart = j + 1;
                }
            }
        }
    }
```

这样就完成了文本的绘制工作，简单地总结就是：分析整体文本—>拆分为段落—>计算整体段落的文本包括Span的测量信息—>对文本进行折行—>根据最终行数把文本测量信息保存—>绘制文本的行背景—>判断并获取文本种的Span和Emoji图像—>绘制最终的文本和图像。当然我们省略了一部分内容，比如段落文本方向，单行的文本排版方向的计算，实际的处理要更为复杂。

接下来我们来看一下在测量过程中出现的FontMetrics，这是一个Paint的静态内部类。主要用来储存文字排版的Y轴相关信息。内部仅包含ascent、descent、top、bottom、leading五个数值。如下图:

 ![1339061786_4121](https://raw.githubusercontent.com/7heaven/AndroidSdkSourceAnalysis/master/article/images/fontmetrics.gif)

除了leading以外，其他的数值都是相对于每一行的baseline的，也就是说其他的数值需要加上对应行的baseline才能得到最终真实的坐标。



## 6.TextView接收软键盘输入

***

Android上的标准文本编辑控件是EditText，而EditText对软键盘输入的处理，却是在TextView内部实现的。Android为所有的View预留了一个接收软键盘输入的接口类，叫InputConnection。软键盘以InputConnection为桥梁把文字输入、文字修改、文字删除等传递给View。任意View只要重写onCheckIsTextEditor()并返回true，然后重写onCreateInputConnection(EditorInfo outAttrs)返回一个InputConnection的实例，便可以接收软键盘的输入。TextView的软键盘输入接收，是通过EditableInputConnection类来实现的。

```java
public InputConnection onCreateInputConnection(EditorInfo outAttrs) {
        //判断是否处于可编辑状态
        if (onCheckIsTextEditor() && isEnabled()) {
            mEditor.createInputMethodStateIfNeeded();
          
            //设置输入法相关的信息
            outAttrs.inputType = getInputType();
            if (mEditor.mInputContentType != null) {
                outAttrs.imeOptions = mEditor.mInputContentType.imeOptions;
                outAttrs.privateImeOptions = mEditor.mInputContentType.privateImeOptions;
                outAttrs.actionLabel = mEditor.mInputContentType.imeActionLabel;
                outAttrs.actionId = mEditor.mInputContentType.imeActionId;
                outAttrs.extras = mEditor.mInputContentType.extras;
            } else {
                outAttrs.imeOptions = EditorInfo.IME_NULL;
            }
            if (focusSearch(FOCUS_DOWN) != null) {
                outAttrs.imeOptions |= EditorInfo.IME_FLAG_NAVIGATE_NEXT;
            }
            if (focusSearch(FOCUS_UP) != null) {
                outAttrs.imeOptions |= EditorInfo.IME_FLAG_NAVIGATE_PREVIOUS;
            }
            if ((outAttrs.imeOptions&EditorInfo.IME_MASK_ACTION)
                    == EditorInfo.IME_ACTION_UNSPECIFIED) {
              
                if ((outAttrs.imeOptions&EditorInfo.IME_FLAG_NAVIGATE_NEXT) != 0) {
                    //把软键盘的enter设为下一步
                    outAttrs.imeOptions |= EditorInfo.IME_ACTION_NEXT;
                } else {
                    //把软键盘的enter设为完成
                    outAttrs.imeOptions |= EditorInfo.IME_ACTION_DONE;
                }
                if (!shouldAdvanceFocusOnEnter()) {
                    outAttrs.imeOptions |= EditorInfo.IME_FLAG_NO_ENTER_ACTION;
                }
            }
            if (isMultilineInputType(outAttrs.inputType)) {
                outAttrs.imeOptions |= EditorInfo.IME_FLAG_NO_ENTER_ACTION;
            }
            outAttrs.hintText = mHint;
          
            //判断TextView内部文本是否可编辑
            if (mText instanceof Editable) {
                //返回EditableInputConnection实例
                InputConnection ic = new EditableInputConnection(this);
                outAttrs.initialSelStart = getSelectionStart();
                outAttrs.initialSelEnd = getSelectionEnd();
                outAttrs.initialCapsMode = ic.getCursorCapsMode(getInputType());
                return ic;
            }
        }
        return null;
    }
```

我们再来看一下EditableInputConnection里面的几个主要的方法：

首先是commitText方法，这个方法接收输入法输入的字符并提交给TextView。

```java
public boolean commitText(CharSequence text, int newCursorPosition) {
        //判断TextView是否为空
        if (mTextView == null) {
            return super.commitText(text, newCursorPosition);
        }
        //判断文本是否Span，来自输入法的Span一般只有SuggestionSpan，SuggestionSpan携带了输入法的错别字修正的词
        if (text instanceof Spanned) {
            Spanned spanned = ((Spanned) text);
            SuggestionSpan[] spans = spanned.getSpans(0, text.length(), SuggestionSpan.class);
            mIMM.registerSuggestionSpansForNotification(spans);
        }

        mTextView.resetErrorChangedFlag();
        //提交字符
        boolean success = super.commitText(text, newCursorPosition);
        mTextView.hideErrorIfUnchanged();
        //返回是否成功
        return success;
    }
```

getEditable方法，这个方法并不是InputConnection接口的一部分，而是EditableInputConnection的父类BaseInputConnection的方法，用来获取一个可编辑对象，EditableInputConnection里面的所有修改都针对这个可编辑对象来做。

```java
public Editable getEditable() {
        TextView tv = mTextView;
        if (tv != null) {
            //返回TextView的可编辑对象
            return tv.getEditableText();
        }
        return null;
    }
```

deleteSurroundingText方法，这个方法用来删除光标前后的内容：

```java

public boolean deleteSurroundingText(int beforeLength, int afterLength) {
        if (DEBUG) Log.v(TAG, "deleteSurroundingText " + beforeLength
                + " / " + afterLength);
        final Editable content = getEditable();
        if (content == null) return false;

        //批量删除标记
        beginBatchEdit();
        
        //获取当前已选择的文本的位置
        int a = Selection.getSelectionStart(content);
        int b = Selection.getSelectionEnd(content);

        if (a > b) {
            int tmp = a;
            a = b;
            b = tmp;
        }

        int ca = getComposingSpanStart(content);
        int cb = getComposingSpanEnd(content);
        if (cb < ca) {
            int tmp = ca;
            ca = cb;
            cb = tmp;
        }
        if (ca != -1 && cb != -1) {
            if (ca < a) a = ca;
            if (cb > b) b = cb;
        }

        int deleted = 0;

        //删除光标之前的文本
        if (beforeLength > 0) {
            int start = a - beforeLength;
            if (start < 0) start = 0;
            content.delete(start, a);
            deleted = a - start;
        }
        //删除光标之后的文本
        if (afterLength > 0) {
            b = b - deleted;

            int end = b + afterLength;
            if (end > content.length()) end = content.length();

            content.delete(b, end);
        }
        
        //结束批量编辑
        endBatchEdit();
        
        return true;
    }
```



commitCompletion和commitCorrection方法，即是用来补全单词和修正错别字的方法，这两个方法内部都是调用TextView对应的方法来实现的。

```java
public boolean commitCompletion(CompletionInfo text) {
        if (DEBUG) Log.v(TAG, "commitCompletion " + text);
        mTextView.beginBatchEdit();
        mTextView.onCommitCompletion(text);
        mTextView.endBatchEdit();
        return true;
    }

    @Override
    public boolean commitCorrection(CorrectionInfo correctionInfo) {
        if (DEBUG) Log.v(TAG, "commitCorrection" + correctionInfo);
        mTextView.beginBatchEdit();
        mTextView.onCommitCorrection(correctionInfo);
        mTextView.endBatchEdit();
        return true;
    }
```

## 8.总结

***

一个展示文本+文本编辑器功能的控件需要做的事情很多，要对文本进行排版、处理不同的段落风格、处理段落内的不同emoji和span、进行折行计算，然后还需要做文本编辑、文本选择等。而TextView把这些事情明确分工给不同的类。这样不仅仅把复杂问题拆分成了一个个简单的小功能，同时也大大增加了可扩展性。