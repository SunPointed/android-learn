# tooltips 学习
Simple to use library for android, Enabling to add a tooltip near any view with ease

* [项目地址](https://github.com/tomergoldst/tooltips)

## 使用
```java
    ToolTip.Builder builder = new ToolTip.Builder(this, mTextView, mRootLayout, TIP_TEXT, ToolTip.POSITION_ABOVE);
    builder.setAlign(mAlign);
    mToolTipsManager.show(builder.build());
```
## 重要变量

## 分析
## 1.ToolTip.Builder -> 构造函数
```java
        /**
         *  初始化一些参数
         * @param context context
         * @param anchorView the view which near it we want to put the tip
         * @param root a class extends ViewGroup which the created tip view will be added to
         * @param message message to show
         * @param position  put the tip above / below / left to / right to
         */
        public Builder(Context context, View anchorView, ViewGroup root, String message, @Position int position){
            mContext = context;
            mAnchorView = anchorView;
            mRootViewGroup = root;
            mMessage = message;
            mSpannableMessage = null;
            mPosition = position;
            mAlign = ALIGN_CENTER;
            mOffsetX = 0;
            mOffsetY = 0;
            mArrow = true;
            mBackgroundColor = context.getResources().getColor(R.color.colorBackground);
            mTextColor = context.getResources().getColor(R.color.colorText);
            mTextGravity = GRAVITY_LEFT;
            mTextSize = 14;
        }
```

## 2.ToolTip.Builder -> build
```java
        public ToolTip build(){
            return new ToolTip(this);
        }
```

## 3.ToolTip -> 构造函数
```java
    // 注意ToolTip并没有继承View类
    // 把builder中的参数传入ToolTip中
    ...
```

## 4.ToolTipsManager -> show
```java
    public View show(ToolTip toolTip) {
        // 跳转5
        View tipView = create(toolTip);
        if (tipView == null) {
            return null;
        }

        // 跳转11
        // animate tip visibility
        AnimationUtils.popup(tipView, mAnimationDuration).start();

        return tipView;
    }
```

## 5.ToolTipsManager -> create
```java
    private View create(ToolTip toolTip) {

        if (toolTip.getAnchorView() == null) {
            Log.e(TAG, "Unable to create a tip, anchor view is null");
            return null;
        }

        if (toolTip.getRootView() == null) {
            Log.e(TAG, "Unable to create a tip, root layout is null");
            return null;
        }

        // 保证一个anchor只有一个tipView，如果已经有了tipView，就直接复用
        // only one tip is allowed near an anchor view at the same time, thus
        // reuse tip if already exist
        if (mTipsMap.containsKey(toolTip.getAnchorView().getId())) {
            return mTipsMap.get(toolTip.getAnchorView().getId());
        }

        // 跳转6
        // init tip view parameters
        TextView tipView = createTipView(toolTip);

        // 如果是从右往左书写的语言，设置ToolTips的展示方向，默认为从左往右
        // on RTL languages replace sides
        if (UiUtils.isRtl()) {
            switchToolTipSidePosition(toolTip);
        }

        // 跳转7 设置toolTip的背景
        // set tool tip background / shape
        ToolTipBackgroundConstructor.setBackground(tipView, toolTip);

        // 添加tipView到anchor所在的viewgroup
        // add tip to root layout
        toolTip.getRootView().addView(tipView);

        // 跳转12 把tipView放到相应的位置
        // find where to position the tool tip
        Point p = ToolTipCoordinatesFinder.getCoordinates(tipView, toolTip);
        // move tip view to correct position
        moveTipToCorrectPosition(tipView, p);

        // set dismiss on click
        tipView.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                // dismiss最终会调用TipListener的onTipDismissed方法，用户可以在tips消失时做一些操作
                dismiss(view, true);
            }
        });

        // 复用相关
        // bind tipView with anchorView id
        int anchorViewId = toolTip.getAnchorView().getId();
        tipView.setTag(anchorViewId);

        // enter tip to map by 'anchorView' id
        mTipsMap.put(anchorViewId, tipView);

        return tipView;

    }
```

## 6.ToolTipsManager -> createTipView
```java
    // 设置textView的相应属性
    @NonNull
    private TextView createTipView(ToolTip toolTip) {
        TextView tipView = new TextView(toolTip.getContext());
        tipView.setTextColor(toolTip.getTextColor());
        tipView.setTextSize(toolTip.getTextSize());
        tipView.setText(toolTip.getMessage() != null ? toolTip.getMessage() : toolTip.getSpannableMessage());
        tipView.setVisibility(View.INVISIBLE);
        tipView.setGravity(toolTip.getTextGravity());
        // 在android版本大于等于Build.VERSION_CODES.LOLLIPOP时，使用高度
        setTipViewElevation(tipView, toolTip);
        return tipView;
    }
```

## 7.ToolTipBackgroundConstructor -> setBackground
```java
    static void setBackground(View tipView, ToolTip toolTip) {

        //没有箭头的话，直接设置背景就好，默认为一个.9的png图片
        // show tool tip without arrow. no need to continue
        if (toolTip.hideArrow()) {
            setToolTipNoArrowBackground(tipView, toolTip.getBackgroundColor());
            return;
        }

        // show tool tip according to requested position
        switch (toolTip.getPosition()) {
            case ToolTip.POSITION_ABOVE:
                // 跳转8 below、left、right同理
                setToolTipAboveBackground(tipView, toolTip);
                break;
            case ToolTip.POSITION_BELOW:
                setToolTipBelowBackground(tipView, toolTip);
                break;
            case ToolTip.POSITION_LEFT_TO:
                setToolTipLeftToBackground(tipView, toolTip.getBackgroundColor());
                break;
            case ToolTip.POSITION_RIGHT_TO:
                setToolTipRightToBackground(tipView, toolTip.getBackgroundColor());
                break;
        }

    }
```

## 8.ToolTipBackgroundConstructor -> setToolTipAboveBackground
```java
    private static void setToolTipAboveBackground(View tipView, ToolTip toolTip) {
        switch (toolTip.getAlign()) {
            case ToolTip.ALIGN_CENTER:
                // 跳转9 主要是设置.9png获得的drawable的颜色
                setTipBackground(tipView, R.drawable.tooltip_arrow_down, toolTip.getBackgroundColor());
                break;
            case ToolTip.ALIGN_LEFT:
                setTipBackground(tipView,
                        !UiUtils.isRtl() ?
                                R.drawable.tooltip_arrow_down_left :
                                R.drawable.tooltip_arrow_down_right
                        , toolTip.getBackgroundColor());
                break;
            case ToolTip.ALIGN_RIGHT:
                setTipBackground(tipView,
                        !UiUtils.isRtl() ?
                                R.drawable.tooltip_arrow_down_right :
                                R.drawable.tooltip_arrow_down_left
                        , toolTip.getBackgroundColor());
                break;
        }
    }
```

## 9.ToolTipBackgroundConstructor -> setTipBackground
```java
   private static void setTipBackground(View tipView, int drawableRes, int color){
        // 跳转10
        Drawable paintedDrawable = getTintedDrawable(tipView.getContext(),
                drawableRes, color);
        // 用paintedDrawable设置tipView的background
        setViewBackground(tipView, paintedDrawable);
    } 
```

## 10.ToolTipBackgroundConstructor -> getTintedDrawable
```java
    private static Drawable getTintedDrawable(Context context, int drawableRes, int color){
        Drawable drawable;
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            drawable = context.getResources().getDrawable(drawableRes, null);
            if (drawable != null) {
                drawable.setTint(color);
            }
        } else {
            drawable = context.getResources().getDrawable(drawableRes);
            if (drawable != null) {
                drawable.setColorFilter(color, PorterDuff.Mode.SRC_ATOP);
            }
        }
        return drawable;
    }
```

## 11.AnimationUtils 使用属性动画来控制view的展示和消失
```java
    class AnimationUtils {

    static ObjectAnimator popup(final View view, final long duration) {
        view.setAlpha(0);
        view.setVisibility(View.VISIBLE);

        ObjectAnimator popup = ObjectAnimator.ofPropertyValuesHolder(view,
                PropertyValuesHolder.ofFloat("alpha", 0f, 1f),
                PropertyValuesHolder.ofFloat("scaleX", 0f, 1f),
                PropertyValuesHolder.ofFloat("scaleY", 0f, 1f));
        popup.setDuration(duration);
        popup.setInterpolator(new OvershootInterpolator());

        return popup;
    }

    static ObjectAnimator popout(final View view, final long duration, final AnimatorListenerAdapter animatorListenerAdapter) {
        ObjectAnimator popout = ObjectAnimator.ofPropertyValuesHolder(view,
                PropertyValuesHolder.ofFloat("alpha", 1f, 0f),
                PropertyValuesHolder.ofFloat("scaleX", 1f, 0f),
                PropertyValuesHolder.ofFloat("scaleY", 1f, 0f));
        popout.setDuration(duration);
        popout.addListener(new AnimatorListenerAdapter() {
            @Override
            public void onAnimationEnd(Animator animation) {
                super.onAnimationEnd(animation);
                view.setVisibility(View.GONE);
                if (animatorListenerAdapter != null) {
                    animatorListenerAdapter.onAnimationEnd(animation);
                }
            }
        });
        popout.setInterpolator(new AnticipateOvershootInterpolator());

        return popout;
    }
}
```

## 12.ToolTipCoordinatesFinder -> getCoordinates
```java
    /**
     * return the top left coordinates for positioning the tip
     * 
     * @param tipView - the newly created tip view
     * @param tooltip - tool tip object
     * @return point
     */
    static Point getCoordinates(final TextView tipView, ToolTip tooltip) {
        Point point = new Point();
        // Coordinates是一个封装left、right、top、bottom位置的类，这里两个Coordinates分别保存anchor和viewgroup的位置信息
        final Coordinates anchorViewCoordinates = new Coordinates(tooltip.getAnchorView());
        final Coordinates rootCoordinates = new Coordinates(tooltip.getRootView());

        tipView.measure(ViewGroup.LayoutParams.WRAP_CONTENT, ViewGroup.LayoutParams.WRAP_CONTENT);

        switch (tooltip.getPosition()) {
            case ToolTip.POSITION_ABOVE:
                // 跳转13 其余三个类似
                point = getPositionAbove(tipView, tooltip,
                        anchorViewCoordinates, rootCoordinates);
                break;
            case ToolTip.POSITION_BELOW:
                point = getPositionBelow(tipView, tooltip,
                        anchorViewCoordinates, rootCoordinates);
                break;
            case ToolTip.POSITION_LEFT_TO:
                point = getPositionLeftTo(tipView, tooltip,
                        anchorViewCoordinates, rootCoordinates);
                break;
            case ToolTip.POSITION_RIGHT_TO:
                point = getPositionRightTo(tipView, tooltip,
                        anchorViewCoordinates, rootCoordinates);
                break;
        }

        // add user defined offset values
        point.x += UiUtils.isRtl() ? -tooltip.getOffsetX() : tooltip.getOffsetX();
        point.y += tooltip.getOffsetY();

        // coordinates retrieved are relative to 0,0 of the root layout
        // added view to root is subject to root padding
        // we need to subtract the top and left padding of root from coordinates. to adjust
        // top left tip coordinates
        point.x -= tooltip.getRootView().getPaddingLeft();
        point.y -= tooltip.getRootView().getPaddingTop();

        return point;

    }
```

## 13.ToolTipCoordinatesFinder -> getPositionAbove
```java
    private static Point getPositionAbove(TextView tipView, ToolTip toolTip,
                                          Coordinates anchorViewCoordinates, Coordinates rootLocation) {
        Point point = new Point();
        // getXOffset根据toolTip.getAlign()获取相应的offset
        point.x = anchorViewCoordinates.left + getXOffset(tipView, toolTip);
        // 各个adjust方法保证tipView不超过viewgroup，Align不同计算策略不同
        if (toolTip.alignedCenter()) {
            AdjustHorizontalCenteredOutOfBounds(tipView, toolTip.getRootView(), point, rootLocation);
        } else if (toolTip.alignedLeft()){
            AdjustHorizontalLeftAlignmentOutOfBounds(tipView, toolTip.getRootView(), point, anchorViewCoordinates, rootLocation);
        } else if (toolTip.alignedRight()){
            AdjustHorizotalRightAlignmentOutOfBounds(tipView, toolTip.getRootView(), point, anchorViewCoordinates, rootLocation);
        }
        point.y = anchorViewCoordinates.top - tipView.getMeasuredHeight();
        return point;
    }
```

分析到此，大致流程都已经了解了，tooltips最巧妙的地方是用.9png图片实现了textView的三角效果（这种骚东西可以省去不少时间啊），如下：

----------
![image](./img/tooltips_question_1.png)