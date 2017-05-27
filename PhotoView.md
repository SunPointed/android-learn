# PhotoView 学习
PhotoView aims to help produce an easily usable implementation of a zooming Android ImageView.

* [项目地址](https://github.com/chrisbanes/PhotoView)

## 重要的类
###  1.GestureDetector
Detects various gestures and events using the supplied MotionEvents. The GestureDetector.OnGestureListener callback will notify users when a particular motion event has occurred. This class should only be used with MotionEvents reported via touch (don't use for trackball events). To use this class:

- Create an instance of the GestureDetector for your View
- In the onTouchEvent(MotionEvent) method ensure you call onTouchEvent(MotionEvent). The methods defined in your callback will be executed when the events occur
- If listening for onContextClick(MotionEvent) you must call onGenericMotionEvent(MotionEvent) in onGenericMotionEvent(MotionEvent)

### 2.ScaleGestureDetector
Detects scaling transformation gestures using the supplied MotionEvents. The ScaleGestureDetector.OnScaleGestureListener callback will notify users when a particular gesture event has occurred. This class should only be used with MotionEvents reported via touch. To use this class:

- Create an instance of the ScaleGestureDetector for your View
- In the onTouchEvent(MotionEvent) method ensure you call onTouchEvent(MotionEvent). The methods defined in your callback will be executed when the events occur

### 3.VelocityTracker
Helper for tracking the velocity of touch events, for implementing flinging and other such gestures. Use obtain() to retrieve a new instance of the class when you are going to begin tracking. Put the motion events you receive into it with addMovement(MotionEvent). When you want to determine the velocity call computeCurrentVelocity(int) and then call getXVelocity(int) and getYVelocity(int) to retrieve the velocity for each pointer id.

## 要点
#### 1.PhotoView在PhotoViewAttacher的构造函数中被setOnTouchListener,事件是从此处开始分发的，关注PhotoViewAttacher实现View.OnTouchListener接口的onTouch方法
```java
    imageView.setOnTouchListener(this);
```
-----------------------
#### 2.PhotoViewAttacher初始化了CustomGestureDetector,关注PhotoViewAttacher实现OnGestureListener的onDrag，onFling，onScale方法在CustomGestureDetector如何使用
```java
    mScaleDragDetector = new CustomGestureDetector(imageView.getContext(), this);
```
-----------------------
#### 3.PhotoViewAttacher的变量
    mBaseMatrix->保存imageView中drawble的初始矩阵信息，稳定
    mSuppMatrix->保存作用于imageView中drawble的矩阵变化信息
    mBaseRotation->默认为0，需手动设置，所以不考虑旋转
    mDrawMatrix->中间变量，由mBaseMatrix和mSuppMatrix计算得到，imageView中drawble每次变化的最终矩阵
-----------------------
## 流程分析
#### 1.PhotoViewAttacher的onTouch
```java
    @Override
    public boolean onTouch(View v, MotionEvent ev) {
        boolean handled = false;

        // mZoomEnabled最开始为true,如果iamgeView没有drawable直接返回false
        if (mZoomEnabled && Util.hasDrawable((ImageView) v)) {
            switch (ev.getAction()) {
                case MotionEvent.ACTION_DOWN:
                    ViewParent parent = v.getParent();
                    // 屏蔽父控件的interceptTouchEvent方法                
                    if (parent != null) {
                        parent.requestDisallowInterceptTouchEvent(true);
                    }

                    // 停止fling，因为用户又touch屏幕了
                    cancelFling();
                    break;

                case MotionEvent.ACTION_CANCEL:
                case MotionEvent.ACTION_UP:
                    // If the user has zoomed less than min scale, zoom back
                    // to min scale
                    if (getScale() < mMinScale) {
                        RectF rect = getDisplayRect();
                        if (rect != null) {
                            v.post(new AnimatedZoomRunnable(getScale(), mMinScale,
                                    rect.centerX(), rect.centerY()));
                            handled = true;
                        }
                    }
                    break;
            }

            // 处理缩放
            if (mScaleDragDetector != null) {
                // 判断滑动手势是否正在进行，实际调用ScaleGestureDetector的isInProgress方法
                boolean wasScaling = mScaleDragDetector.isScaling();
                // 判断是否拖动，最初为false
                boolean wasDragging = mScaleDragDetector.isDragging();

                // 把事件交给CustomGestureDetector的onTouchEvent方法处理，执行该行后，handled是一直为true的
                handled = mScaleDragDetector.onTouchEvent(ev);

                // 如果仍然处于缩放或拖动，就继续屏蔽父控件的interceptTouchEvent方法
                boolean didntScale = !wasScaling && !mScaleDragDetector.isScaling();
                boolean didntDrag = !wasDragging && !mScaleDragDetector.isDragging();

                mBlockParentIntercept = didntScale && didntDrag;
            }

            // 检查用户是否双击，把事件交给GestureDetector的onTouchEvent方法处理
            if (mGestureDetector != null && mGestureDetector.onTouchEvent(ev)) {
                handled = true;
            }

        }

        return handled;
    }
```
####此处重要的方法是CustomGestureDetector的onTouchEvent（跳转2），GestureDetector的onTouchEvent（跳转4），
-----------------------
#### 2.CustomGestureDetector的onTouchEvent
```java
    public boolean onTouchEvent(MotionEvent ev) {
        try {
            // 此处的mDEtector即为ScaleGestureDetector,这里的会执行到PhotoViewAttacher实现OnGestureListener的onScale方法，在mDEtector设置的ScaleGestureDetector.OnScaleGestureListener中会调用onScale方法，mDEtector在CustomGestureDetector的构造函数中初始化
            mDetector.onTouchEvent(ev);
            // processTouchEvent
            return processTouchEvent(ev);
        } catch (IllegalArgumentException e) {
            // Fix for support lib bug, happening when onDestroy is called
            return true;
        }
    }
```
####此处重要的方法是mDetector击ScaleGestureDetector的onTouchEvent（跳转3），CustomGestureDetector的processTouchEvent（跳转5）
-----------------------
#### 3.CustomGestureDetector的构造函数，在mDetector设置了mScaleListener，所以在mDetector的onTouchEvent方法中会调用mScaleListener的onScale，最终调用mListener的onScale，而mListener就是实现了OnGestureListener接口的PhotoViewAttacher
```java
    CustomGestureDetector(Context context, OnGestureListener listener) {
        final ViewConfiguration configuration = ViewConfiguration
                .get(context);
        mMinimumVelocity = configuration.getScaledMinimumFlingVelocity();
        mTouchSlop = configuration.getScaledTouchSlop();

        // 这个listener就是实现了OnGestureListener接口的PhotoViewAttacher
        mListener = listener;
        ScaleGestureDetector.OnScaleGestureListener mScaleListener = new ScaleGestureDetector.OnScaleGestureListener() {

            @Override
            public boolean onScale(ScaleGestureDetector detector) {
                float scaleFactor = detector.getScaleFactor();

                if (Float.isNaN(scaleFactor) || Float.isInfinite(scaleFactor))
                    return false;

                // 此处会调用PhotoViewAttacher的onScale方法，计算所方式矩阵的变化，跳转14
                mListener.onScale(scaleFactor,
                        detector.getFocusX(), detector.getFocusY());
                return true;
            }

            @Override
            public boolean onScaleBegin(ScaleGestureDetector detector) {
                return true;
            }

            @Override
            public void onScaleEnd(ScaleGestureDetector detector) {
                // NO-OP
            }
        };
        // 初始化mDetector
        mDetector = new ScaleGestureDetector(context, mScaleListener);
    }
```
####此处我们回到PhotoViewAttacher中（跳转6）
-----------------------
#### 4.mGestureDetector的onTouchEvent，mGestureDetector在PhotoViewAttacher的构造函数中初始化，主要处理长按，双击，用户设置的mSingleFlingListener，如下
```java
    mGestureDetector = new GestureDetector(imageView.getContext(), new GestureDetector.SimpleOnGestureListener() {

            // 处理长按事件
            @Override
            public void onLongPress(MotionEvent e) {
                if (mLongClickListener != null) {
                    mLongClickListener.onLongClick(mImageView);
                }
            }
            // 处理用户设置的mSingleFlingListener
            @Override
            public boolean onFling(MotionEvent e1, MotionEvent e2,
                                   float velocityX, float velocityY) {
                if (mSingleFlingListener != null) {
                    if (getScale() > DEFAULT_MIN_SCALE) {
                        return false;
                    }

                    if (MotionEventCompat.getPointerCount(e1) > SINGLE_TOUCH
                            || MotionEventCompat.getPointerCount(e2) > SINGLE_TOUCH) {
                        return false;
                    }

                    return mSingleFlingListener.onFling(e1, e2, velocityX, velocityY);
                }
                return false;
            }
        });

        // 处理双击事件
        mGestureDetector.setOnDoubleTapListener(new GestureDetector.OnDoubleTapListener() {
            // ...
        });
```
-----------------------
#### 5.CustomGestureDetector的processTouchEvent
```java
    private boolean processTouchEvent(MotionEvent ev) {
        final int action = ev.getAction();
        switch (action & MotionEvent.ACTION_MASK) {
            case MotionEvent.ACTION_DOWN:
                // 获取第一个触点作为当前触点
                mActivePointerId = ev.getPointerId(0);

                mVelocityTracker = VelocityTracker.obtain();
                if (null != mVelocityTracker) {
                    mVelocityTracker.addMovement(ev);
                }

                // 第一个触点的位置
                mLastTouchX = getActiveX(ev);
                mLastTouchY = getActiveY(ev);
                // mIsDragging为false
                mIsDragging = false;
                break;
            case MotionEvent.ACTION_MOVE:
                // 计算第一个触点的位移
                final float x = getActiveX(ev);
                final float y = getActiveY(ev);
                final float dx = x - mLastTouchX, dy = y - mLastTouchY;

                if (!mIsDragging) {
                    // 位移大于mTouchSlop时，mIsDragging为true
                    mIsDragging = Math.sqrt((dx * dx) + (dy * dy)) >= mTouchSlop;
                }

                if (mIsDragging) {
                    // mIsDragging为true,调用OnGestureListener的onDrag，PhotoViewAttacher自身实现OnGestureListener接口
                    mListener.onDrag(dx, dy);
                    mLastTouchX = x;
                    mLastTouchY = y;

                    if (null != mVelocityTracker) {
                        mVelocityTracker.addMovement(ev);
                    }
                }
                break;
            case MotionEvent.ACTION_CANCEL:
                mActivePointerId = INVALID_POINTER_ID;
                // Recycle Velocity Tracker
                if (null != mVelocityTracker) {
                    mVelocityTracker.recycle();
                    mVelocityTracker = null;
                }
                break;
            case MotionEvent.ACTION_UP:
                mActivePointerId = INVALID_POINTER_ID;
                if (mIsDragging) {
                    if (null != mVelocityTracker) {
                        mLastTouchX = getActiveX(ev);
                        mLastTouchY = getActiveY(ev);

                        // 计算最后一秒的速率
                        mVelocityTracker.addMovement(ev);
                        mVelocityTracker.computeCurrentVelocity(1000);

                        final float vX = mVelocityTracker.getXVelocity(), vY = mVelocityTracker
                                .getYVelocity();

                        // 如果速率大于mMinimumVelocity，调用OnGestureListener的onFling，PhotoViewAttacher自身实现OnGestureListener接口
                        if (Math.max(Math.abs(vX), Math.abs(vY)) >= mMinimumVelocity) {
                            mListener.onFling(mLastTouchX, mLastTouchY, -vX,
                                    -vY);
                        }
                    }
                }

                // Recycle Velocity Tracker
                if (null != mVelocityTracker) {
                    mVelocityTracker.recycle();
                    mVelocityTracker = null;
                }
                break;
            case MotionEvent.ACTION_POINTER_UP:
                final int pointerIndex = Util.getPointerIndex(ev.getAction());
                final int pointerId = ev.getPointerId(pointerIndex);
                if (pointerId == mActivePointerId) {
                    // 如果第一个触点失效了，选取下一个触点作为当前的触点
                    final int newPointerIndex = pointerIndex == 0 ? 1 : 0;
                    mActivePointerId = ev.getPointerId(newPointerIndex);
                    mLastTouchX = ev.getX(newPointerIndex);
                    mLastTouchY = ev.getY(newPointerIndex);
                }
                break;
        }

        // 保存当前触点的索引， mActivePointerIndex会在getActiveX,getActiveY方法中使用
        mActivePointerIndex = ev
                .findPointerIndex(mActivePointerId != INVALID_POINTER_ID ? mActivePointerId
                        : 0);
        return true;
    }
```
####此处在MotionEvent.ACTION_MOVE中调用mListener.onDrag，在MotionEvent.ACTION_UP调用mListener.onFling（当然都要满足一定的条件），同样mListener就是实现了OnGestureListener接口的PhotoViewAttacher，回到PhotoViewAttacher中（跳转6）
-----------------------
#### 6.PhotoViewAttacher的onScale, onDrag, onFling
###### 6.1 在此之前，先介绍几个PhotoViewAttacher中的方法
######getDrawMatrix，各种事件会将矩阵的变化保存到mSuppMatrix中，而mBaseMatrix保存imageView中drawable原本的矩阵信息，该方法返回变化后的矩阵信息
```java
    private Matrix getDrawMatrix() {
        // 用mDrawMatrix保存mBaseMatrix的信息，mBaseMatrix在updateBaseMatrix方法中保存的是drawable原本的矩阵信息
        mDrawMatrix.set(mBaseMatrix);
        // mDrawMatrix和mSuppMatrix合并，mDrawMatrix中保存的是变化后的矩阵信息
        mDrawMatrix.postConcat(mSuppMatrix);
        return mDrawMatrix;
    }
```

######setImageViewMatrix，完成一次imageView中drawable的变化（缩放、位移）
```java
    private void setImageViewMatrix(Matrix matrix) {
        // 调用ImageView的setImageMatrix将矩阵应用到imageView的drawable上
        mImageView.setImageMatrix(matrix);

        if (mMatrixChangeListener != null) {
            RectF displayRect = getDisplayRect(matrix);
            if (displayRect != null) {
                // 调用用户设置的mMatrixChangeListener
                mMatrixChangeListener.onMatrixChanged(displayRect);
            }
        }
    }
```

######checkMatrixBounds，该函数将drawable因ScaleType产生位移信息保存在mSuppMatrix中
```java
    private boolean checkMatrixBounds() {

        // 通过mapRect方法将getDrawMatrix返回的矩阵应用到mDisplayRect，mDisplayRect设置为iamgeView的drawable的大小
        final RectF rect = getDisplayRect(getDrawMatrix());
        if (rect == null) {
            return false;
        }

        final float height = rect.height(), width = rect.width();
        float deltaX = 0, deltaY = 0;

        // 根据ScaleType获取y轴移动的距离
        final int viewHeight = getImageViewHeight(mImageView);
        // ...

        // 根据ScaleType获取x轴移动的距离
        final int viewWidth = getImageViewWidth(mImageView);
        // ...

        // 将x,y方向上移动的距离保存到mSuppMatrix
        mSuppMatrix.postTranslate(deltaX, deltaY);
        return true;
    }
```

######checkAndDisplayMatrix，上面三个方法的结合使用，onScale,onDrag会用到
```java
    private void checkAndDisplayMatrix() {
        // checkMatrixBounds
        if (checkMatrixBounds()) {
            // getDrawMatrix
            // setImageViewMatrix
            setImageViewMatrix(getDrawMatrix());  
        }
    }
```
-----------------------
###### 6.2 PhotoViewAttacher的onScale方法
```java
    @Override
    public void onScale(float scaleFactor, float focusX, float focusY) {
        if ((getScale() < mMaxScale || scaleFactor < 1f) && (getScale() > mMinScale || scaleFactor > 1f)) {
            if (mScaleChangeListener != null) {
                // 调用用户设置的mScaleChangeListener
                mScaleChangeListener.onScaleChange(scaleFactor, focusX, focusY);
            }
            // 将缩放保存在mSuppMatrix中
            mSuppMatrix.postScale(scaleFactor, scaleFactor, focusX, focusY);
            // PhotoViewAttacher的checkAndDisplayMatrix
            checkAndDisplayMatrix();
        }
    }
```
-----------------------
###### 6.3 PhotoViewAttacher的onFling方法
```java
    @Override
    public void onFling(float startX, float startY, float velocityX,
                        float velocityY) {
        // 此处向主线程post了一个FlingRunnable，FlingRunnable是PhotoViewAttacher的内部类
        mCurrentFlingRunnable = new FlingRunnable(mImageView.getContext());
        mCurrentFlingRunnable.fling(getImageViewWidth(mImageView),
                getImageViewHeight(mImageView), (int) velocityX, (int) velocityY);
        mImageView.post(mCurrentFlingRunnable);
    }
```

######FlingRunnable
```java
    private class FlingRunnable implements Runnable {

        private final OverScroller mScroller;
        private int mCurrentX, mCurrentY;

        // ...

        // fling的条件是当前iamgeView的width或height小于getDisplayRect获得的rect的width或height，简单说就是drawable超出了imageView的范围
        public void fling(int viewWidth, int viewHeight, int velocityX,
                          int velocityY) {
            // ...

            // startX为Math.round(-rect.left)，注意理解负号，首先如果要fling，上面说过了drawable超出了imageView的范围，所以 -rect.left实际上是一个正的值！这个值在0到rect.width() - viewWidth范围之间
            final int startX = Math.round(-rect.left);
            final int minX, maxX, minY, maxY;

            if (viewWidth < rect.width()) {
                minX = 0;
                maxX = Math.round(rect.width() - viewWidth);
            } else {
                minX = maxX = startX;
            }

            // 同见startX的解释
            // ...

            mCurrentX = startX;
            mCurrentY = startY;

            // If we actually can move, fling the scroller
            if (startX != maxX || startY != maxY) {
                mScroller.fling(startX, startY, velocityX, velocityY, minX,
                        maxX, minY, maxY, 0, 0);
            }
        }

        @Override
        public void run() {
            if (mScroller.isFinished()) {
                return; // remaining post that should not be handled
            }

            if (mScroller.computeScrollOffset()) {

                // newX左滑增大，右滑减小， newY上划增大，下划减小，此处不要弄混了触点的x,y值得增大减小与newX，newY的增大减小
                final int newX = mScroller.getCurrX();
                final int newY = mScroller.getCurrY();

                // 左划mCurrentX - newX 为负，左移
                // 右划mCurrentX - newX 为正，右移
                // mCurrentY - newY同理
                mSuppMatrix.postTranslate(mCurrentX - newX, mCurrentY - newY);
                setImageViewMatrix(getDrawMatrix());

                mCurrentX = newX;
                mCurrentY = newY;

                // 不断向主线程post FlingRunnable的当前实例，mScroller完成或调用cancelFling方法才停止
                Compat.postOnAnimation(mImageView, this);
            }
        }
    }
```
-----------------------
###### 6.4 PhotoViewAttacher的onDrag方法
```java
    @Override
    public void onDrag(float dx, float dy) {
        if (mScaleDragDetector.isScaling()) {
            return; // Do not drag if we are already scaling
        }

        // 将拖动保存在mSuppMatrix中
        mSuppMatrix.postTranslate(dx, dy);
        
        checkAndDisplayMatrix();

        // 判断是否继续设置父控件requestDisallowInterceptTouchEvent(false)
        // ...
    }
```
-----------------------
## 其他
PhotoViewAttacher构造函数中为imageView添加了View.OnLayoutChangeListener
```java
    imageView.addOnLayoutChangeListener(this);
```
---
当iamgeView发生变化时
```java
    @Override
    public void onLayoutChange(View v, int left, int top, int right, int bottom, int oldLeft, int oldTop, int oldRight, int oldBottom) {
        // Update our base matrix, as the bounds have changed
        updateBaseMatrix(mImageView.getDrawable());
    }
```
---
PhotoViewAttacher的updateBaseMatrix会将mBaseMatrix重置为单位矩阵
```java
    private void updateBaseMatrix(Drawable drawable) {
        if (drawable == null) {
            return;
        }

        // 获取imageView和drawable的宽高
        // ...

        // 讲mBaseMatrix重置为单位矩阵
        mBaseMatrix.reset();

        final float widthScale = viewWidth / drawableWidth;
        final float heightScale = viewHeight / drawableHeight;

        // 根据不同的ScaleType计算相应的矩阵，计算结果保存在mBaseMatrix中，省略
        // ...

        //重置各个矩阵
        resetMatrix();
    }
```
---
resetMatrix置mSuppMatrix为单位矩阵并用它将用户设置的mBaseRotation作用到drawable上
```java
    private void resetMatrix() {
        // 置mSuppMatrix为单位矩阵
        mSuppMatrix.reset();
        // 将mBaseRotation的旋转保存到mSuppMatrix中，mBaseRotation默认为0.0f
        setRotationBy(mBaseRotation);
        // 将最初的矩阵（最多有旋转信息）作用到imageView上
        setImageViewMatrix(getDrawMatrix());
        checkMatrixBounds();
    }
```