# Slidr 学习
Easily add slide-to-dismiss functionality to your Activity by calling Slidr.attach(this) in your onCreate(..) method.

* [项目地址](https://github.com/r0adkll/Slidr)

## 使用
```java
    public class ExampleActivity extends <Activity|FragmentActivity|ActionBarActivity> {

        @Override
        public void onCreate(Bundle savedInstanceState){
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_example);
            int primary = getResources().getColor(R.color.primaryDark);
            int secondary = getResources().getColor(R.color.secondaryDark);
            Slidr.attach(this, primary, secondary);
        }
    }
```
## 或者
```java
    public class ExampleActivity extends <Activity|FragmentActivity|ActionBarActivity> {

        @Override
        public void onCreate(Bundle savedInstanceState){
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_example);
            Slidr.attach(this);
        }
    }
```

## 分析

### Slidr的attach方法
```java
    public static SlidrInterface attach(Activity activity){
        return attach(activity, -1, -1);
    }
```

```java
    public static SlidrInterface attach(final Activity activity, final int statusBarColor1, final int statusBarColor2){

        // Setup the slider panel and attach it to the decor
        final SliderPanel panel = initSliderPanel(activity, null);

        // Set the panel slide listener for when it becomes closed or opened
        panel.setOnPanelSlideListener(new SliderPanel.OnPanelSlideListener() {

            private final ArgbEvaluator mEvaluator = new ArgbEvaluator();

            @Override
            public void onStateChanged(int state) {

            }

            @Override
            public void onClosed() {
                activity.finish();
                activity.overridePendingTransition(0, 0);
            }

            @Override
            public void onOpened() {

            }

            @TargetApi(Build.VERSION_CODES.LOLLIPOP)
            @Override
            public void onSlideChange(float percent) {
                // Interpolate the statusbar color
                if(Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP &&
                        statusBarColor1 != -1 && statusBarColor2 != -1){
                    int newColor = (int) mEvaluator.evaluate(percent, statusBarColor1, statusBarColor2);
                    activity.getWindow().setStatusBarColor(newColor);
                }
            }
        });

        // Return the lock interface
        return initInterface(panel);
    }
```

```java
    public static SlidrInterface attach(final Activity activity, final SlidrConfig config){

        // Setup the slider panel and attach it to the decor
        final SliderPanel panel = initSliderPanel(activity, config);

        // Set the panel slide listener for when it becomes closed or opened
        panel.setOnPanelSlideListener(new SliderPanel.OnPanelSlideListener() {

            private final ArgbEvaluator mEvaluator = new ArgbEvaluator();

            @Override
            public void onStateChanged(int state) {
                if(config.getListener() != null){
                    config.getListener().onSlideStateChanged(state);
                }
            }

            @Override
            public void onClosed() {
                if(config.getListener() != null){
                    config.getListener().onSlideClosed();
                }

                activity.finish();
                activity.overridePendingTransition(0, 0);
            }

            @Override
            public void onOpened() {
                if(config.getListener() != null){
                    config.getListener().onSlideOpened();
                }
            }

            @TargetApi(Build.VERSION_CODES.LOLLIPOP)
            @Override
            public void onSlideChange(float percent) {
                // Interpolate the statusbar color
                // TODO: Add support for KitKat
                if(Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP &&
                        config.areStatusBarColorsValid()){

                    int newColor = (int) mEvaluator.evaluate(percent, config.getPrimaryColor(),
                            config.getSecondaryColor());

                    activity.getWindow().setStatusBarColor(newColor);
                }

                if(config.getListener() != null){
                    config.getListener().onSlideChange(percent);
                }
            }
        });

        // Return the lock interface
        return initInterface(panel);
    }
```
构造SliderPanel并为其setOnPanelSlideListener，如果SlidrConfig不为null，在SliderPanel.OnPanelSlideListener中将事件交给SlidrConfig的SlidrListener处理。

### Slidr的initInterface方法
```java
    private static SlidrInterface initInterface(final SliderPanel panel) {
        // Setup the lock interface
        SlidrInterface slidrInterface = new SlidrInterface() {
            @Override
            public void lock() {
                panel.lock();
            }

            @Override
            public void unlock() {
                panel.unlock();
            }
        };

        // Return the lock interface
        return slidrInterface;
    }
```
返回一个SlidrInterface用来执行SliderPanel的lock与unlock，lock后就没有Slidr的效果了

### Slidr的initSliderPanel方法
```java
   private static SliderPanel initSliderPanel(final Activity activity, final SlidrConfig config) {
        // Hijack the decorview
        ViewGroup decorView = (ViewGroup)activity.getWindow().getDecorView();
        View oldScreen = decorView.getChildAt(0);
        decorView.removeViewAt(0);

        // Setup the slider panel and attach it to the decor
        SliderPanel panel = new SliderPanel(activity, oldScreen, config);
        panel.setId(R.id.slidable_panel);
        oldScreen.setId(R.id.slidable_content);
        panel.addView(oldScreen);
        decorView.addView(panel, 0);
        return panel;
    } 
```
获取activity的decorView，在原有视图和decorView之间加了一个SliderPanel

### SliderPanel的构造函数
```java
    public SliderPanel(Context context, View decorView, SlidrConfig config){
        super(context);
        mDecorView = decorView;
        mConfig = (config == null ? new SlidrConfig.Builder().build() : config);
        init();
    }
```


init()
```java
    private void init(){
        mScreenWidth = getResources().getDisplayMetrics().widthPixels;

        final float density = getResources().getDisplayMetrics().density;
        final float minVel = MIN_FLING_VELOCITY * density;

        ViewDragHelper.Callback callback;
        switch (mConfig.getPosition()){
            case LEFT:
                callback = mLeftCallback;
                mEdgePosition = ViewDragHelper.EDGE_LEFT;
                break;
            case RIGHT:
                callback = mRightCallback;
                mEdgePosition = ViewDragHelper.EDGE_RIGHT;
                break;
            case TOP:
                callback = mTopCallback;
                mEdgePosition = ViewDragHelper.EDGE_TOP;
                break;
            case BOTTOM:
                callback = mBottomCallback;
                mEdgePosition = ViewDragHelper.EDGE_BOTTOM;
                break;
            case VERTICAL:
                callback = mVerticalCallback;
                mEdgePosition = ViewDragHelper.EDGE_TOP | ViewDragHelper.EDGE_BOTTOM;
                break;
            case HORIZONTAL:
                callback = mHorizontalCallback;
                mEdgePosition = ViewDragHelper.EDGE_LEFT | ViewDragHelper.EDGE_RIGHT;
                break;
            default:
                callback = mLeftCallback;
                mEdgePosition = ViewDragHelper.EDGE_LEFT;
        }

        mDragHelper = ViewDragHelper.create(this, mConfig.getSensitivity(), callback);
        mDragHelper.setMinVelocity(minVel);
        mDragHelper.setEdgeTrackingEnabled(mEdgePosition);

        // false to only allow one child view to be the target of any MotionEvent received by this ViewGroup.
        ViewGroupCompat.setMotionEventSplittingEnabled(this, false);

        // Setup the dimmer view
        mDimView = new View(getContext());
        mDimView.setBackgroundColor(mConfig.getScrimColor());
        mDimView.setAlpha(mConfig.getScrimStartAlpha());

        // Add the dimmer view to the layout
        addView(mDimView);

        /*
         * This is so we can get the height of the view and
         * ignore the system navigation that would be included if we
         * retrieved this value from the DisplayMetrics
         */
        post(new Runnable() {
            @Override
            public void run() {
                mScreenHeight = getHeight();
            }
        });

    }
```
根据SlidrConfig设置ViewDragHelper.Callback，生成ViewDragHelper对象，添加mDimView

### mLeftCallback，其余的ViewDragHelper.Callback类似
```java
    /**
     * A Callback is used as a communication channel with the ViewDragHelper back to the
     * parent view using it. on methods are invoked on siginficant events and several
     * accessor methods are expected to provide the ViewDragHelper with more information
     * about the state of the parent view upon request. The callback also makes decisions
     * governing the range and draggability of child views.
     */
    private ViewDragHelper.Callback mLeftCallback = new ViewDragHelper.Callback() {

        // Called when the user's input indicates that they want to capture the given child view with the pointer indicated by pointerId. The callback should return true if the user is permitted to drag the given view with the indicated pointer. 

        // 此处mDecorView是原来的根布局，即oldScreen
        @Override
        public boolean tryCaptureView(View child, int pointerId) {
            boolean edgeCase = !mConfig.isEdgeOnly() || mDragHelper.isEdgeTouched(mEdgePosition, pointerId);
            return child.getId() == mDecorView.getId() && edgeCase;
        }

        // Restrict the motion of the dragged child view along the horizontal axis. The default implementation does not allow horizontal motion; the extending class must override this method and provide the desired clamping.
        @Override
        public int clampViewPositionHorizontal(View child, int left, int dx) {
            return clamp(left, 0, mScreenWidth);
        }

        // Return the magnitude of a draggable child view's horizontal range of motion in pixels. This method should return 0 for views that cannot move horizontally.
        @Override
        public int getViewHorizontalDragRange(View child) {
            return mScreenWidth;
        }

        // 当view被释放时调用，此处的作用是满足条件时把view恢复原位，需要在ViewGroup中重写computeScroll
        /**
         *   @Override
         *   public void computeScroll() {
         *       super.computeScroll();
         *       if(mDragHelper.continueSettling(true)){
         *           ViewCompat.postInvalidateOnAnimation(this);
         *       }
         *   }
         */
        @Override
        public void onViewReleased(View releasedChild, float xvel, float yvel) {
            super.onViewReleased(releasedChild, xvel, yvel);

            int left = releasedChild.getLeft();
            int settleLeft = 0;
            int leftThreshold = (int) (getWidth() * mConfig.getDistanceThreshold());
            boolean isVerticalSwiping = Math.abs(yvel) > mConfig.getVelocityThreshold();

            if(xvel > 0){

                if(Math.abs(xvel) > mConfig.getVelocityThreshold() && !isVerticalSwiping){
                    settleLeft = mScreenWidth;
                }else if(left > leftThreshold){
                    settleLeft = mScreenWidth;
                }

            }else if(xvel == 0){
                if(left > leftThreshold){
                    settleLeft = mScreenWidth;
                }
            }

            // Settle the captured view at the given (left, top) position. The appropriate velocity from prior motion will be taken into account. If this method returns true, the caller should invoke {@link #continueSettling(boolean)} on each subsequent frame to continue the motion until it returns false. If this method returns false there is no further work to do to complete the movement.
            // 按照上面的调用，CapturedView会恢复到settleLeft, releasedChild.getTop()的位置
            mDragHelper.settleCapturedViewAt(settleLeft, releasedChild.getTop());
            invalidate();
        }

        // Called when the captured view's position changes as the result of a drag or settle
        // 主要是用来改变dimView的透明度以及调用OnPanelSlideListener的方法
        @Override
        public void onViewPositionChanged(View changedView, int left, int top, int dx, int dy) {
            super.onViewPositionChanged(changedView, left, top, dx, dy);
            float percent = 1f - ((float)left / (float)mScreenWidth);

            if(mListener != null) mListener.onSlideChange(percent);

            // Update the dimmer alpha
            applyScrim(percent);
        }

        // Called when the drag state changes. See the <code>STATE_*</code> constants for more information
        // 调用OnPanelSlideListener的方法,其中Slidr的attach方法传入的OnPanelSlideListener的onClosed方法调用了activity的finish事件，就实现了右滑关闭的效果
        @Override
        public void onViewDragStateChanged(int state) {
            super.onViewDragStateChanged(state);
            if(mListener != null) mListener.onStateChanged(state);
            switch (state){
                // A view is not currently being dragged or animating as a result of a fling/snap
                case ViewDragHelper.STATE_IDLE:
                    if(mDecorView.getLeft() == 0){
                        // State Open
                        if(mListener != null) mListener.onOpened();
                    }else{
                        // State Closed
                        if(mListener != null) mListener.onClosed();
                    }
                    break;
                // A view is currently being dragged. The position is currently changing as a result of user input or simulated user input.
                case ViewDragHelper.STATE_DRAGGING:
                    break;
                // A view is currently settling into place as a result of a fling or predefined non-interactive motion.
                case ViewDragHelper.STATE_SETTLING:
                    break;
            }
        }

    };
```

