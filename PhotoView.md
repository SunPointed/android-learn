# PhotoView 学习
PhotoView aims to help produce an easily usable implementation of a zooming Android ImageView.

<!-- [![](https://jitpack.io/v/chrisbanes/PhotoView.svg)](https://jitpack.io/#chrisbanes/PhotoView) -->

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

## 流程分析

### 1.PhotoView
继承自ImageView，内部逻辑交给PhotoViewAttacher类的实例attacher处理

### 2.PhotoViewAttacher
实现View.OnTouchListener，View.OnLayoutChangeListener，OnGestureListener

1.构造函数如下
```java
public PhotoViewAttacher(ImageView imageView) {
        mImageView = imageView;
        //为iamgeView添加监听事件
        imageView.setOnTouchListener(this);
        imageView.addOnLayoutChangeListener(this);

        //...

        // Create Gesture Detectors...
        mScaleDragDetector = new CustomGestureDetector(imageView.getContext(), this);

        mGestureDetector = new GestureDetector(imageView.getContext(), new GestureDetector.SimpleOnGestureListener() {

            // 处理长按事件
            @Override
            public void onLongPress(MotionEvent e) {
                if (mLongClickListener != null) {
                    mLongClickListener.onLongClick(mImageView);
                }
            }
            // 处理Fling事件
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
            @Override
            public boolean onSingleTapConfirmed(MotionEvent e) {
                if (mOnClickListener != null) {
                    mOnClickListener.onClick(mImageView);
                }
                final RectF displayRect = getDisplayRect();

                final float x = e.getX(), y = e.getY();

                if (mViewTapListener != null) {
                    mViewTapListener.onViewTap(mImageView, x, y);
                }

                if (displayRect != null) {

                    // Check to see if the user tapped on the photo
                    if (displayRect.contains(x, y)) {

                        float xResult = (x - displayRect.left)
                                / displayRect.width();
                        float yResult = (y - displayRect.top)
                                / displayRect.height();

                        if (mPhotoTapListener != null) {
                            mPhotoTapListener.onPhotoTap(mImageView, xResult, yResult);
                        }
                        return true;
                    } else {
                        if (mOutsidePhotoTapListener != null) {
                            mOutsidePhotoTapListener.onOutsidePhotoTap(mImageView);
                        }
                    }
                }
                return false;
            }

            @Override
            public boolean onDoubleTap(MotionEvent ev) {
                try {
                    float scale = getScale();
                    float x = ev.getX();
                    float y = ev.getY();

                    if (scale < getMediumScale()) {
                        setScale(getMediumScale(), x, y, true);
                    } else if (scale >= getMediumScale() && scale < getMaximumScale()) {
                        setScale(getMaximumScale(), x, y, true);
                    } else {
                        setScale(getMinimumScale(), x, y, true);
                    }
                } catch (ArrayIndexOutOfBoundsException e) {
                    // Can sometimes happen when getX() and getY() is called
                }

                return true;
            }

            @Override
            public boolean onDoubleTapEvent(MotionEvent e) {
                // Wait for the confirmed onDoubleTap() instead
                return false;
            }
        });
    }
```

2.先看一下onTouch方法的处理
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

                    // If we're flinging, and the user presses down, cancel
                    // fling
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

                // 把事件交给CustomGestureDetector的onTouchEvent方法处理，执行该行后，handled是一直为true的，跳转3
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

3.CustomGestureDetector的onTouchEvent方法，处理缩放
```java
public boolean onTouchEvent(MotionEvent ev) {
        try {
            // 此处的mDEtector即为ScaleGestureDetector
            mDetector.onTouchEvent(ev);
            // processTouchEvent，跳转到4
            return processTouchEvent(ev);
        } catch (IllegalArgumentException e) {
            // Fix for support lib bug, happening when onDestroy is called
            return true;
        }
    }
```

4.CustomGestureDetector的processTouchEvent方法，此方法始终返回true
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
                    // mIsDragging为true,调用OnGestureListener的onDrag，PhotoViewAttacher自身实现OnGestureListener接口，跳转5
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

                        // 如果速率大于mMinimumVelocity，调用OnGestureListener的onFling，PhotoViewAttacher自身实现OnGestureListener接口，跳转11
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

5.PhotoViewAttacher实现OnGestureListener的onDrag，
```java
    @Override
    public void onDrag(float dx, float dy) {
        if (mScaleDragDetector.isScaling()) {
            return; // Do not drag if we are already scaling
        }

        // mSuppMatrix初始化为new Matrix()
        mSuppMatrix.postTranslate(dx, dy);
        // 跳转6
        checkAndDisplayMatrix();

        /*
         * Here we decide whether to let the ImageView's parent to start taking
         * over the touch event.
         *
         * First we check whether this function is enabled. We never want the
         * parent to take over if we're scaling. We then check the edge we're
         * on, and the direction of the scroll (i.e. if we're pulling against
         * the edge, aka 'overscrolling', let the parent take over).
         */
        ViewParent parent = mImageView.getParent();
        // mAllowParentInterceptOnEdge默认为true 
        if (mAllowParentInterceptOnEdge && !mScaleDragDetector.isScaling() && !mBlockParentIntercept) {
            if (mScrollEdge == EDGE_BOTH
                    || (mScrollEdge == EDGE_LEFT && dx >= 1f)
                    || (mScrollEdge == EDGE_RIGHT && dx <= -1f)) {
                if (parent != null) {
                    parent.requestDisallowInterceptTouchEvent(false);
                }
            }
        } else {
            if (parent != null) {
                parent.requestDisallowInterceptTouchEvent(true);
            }
        }
    }
```

6.PhotoViewAttacher的checkAndDisplayMatrix
```java
private void checkAndDisplayMatrix() {
    // checkMatrixBounds,跳转7
    if (checkMatrixBounds()) {
        // getDrawMatrix，跳转8
        // setImageViewMatrix，跳转9
        setImageViewMatrix(getDrawMatrix());  
    }
}
```

7.PhotoViewAttacher的checkMatrixBounds
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
        if (height <= viewHeight) {
            switch (mScaleType) {
                case FIT_START:
                    deltaY = -rect.top;
                    break;
                case FIT_END:
                    deltaY = viewHeight - height - rect.top;
                    break;
                default:
                    deltaY = (viewHeight - height) / 2 - rect.top;
                    break;
            }
        } else if (rect.top > 0) {
            deltaY = -rect.top;
        } else if (rect.bottom < viewHeight) {
            deltaY = viewHeight - rect.bottom;
        }

        // 根据ScaleType获取x轴移动的距离
        final int viewWidth = getImageViewWidth(mImageView);
        if (width <= viewWidth) {
            switch (mScaleType) {
                case FIT_START:
                    deltaX = -rect.left;
                    break;
                case FIT_END:
                    deltaX = viewWidth - width - rect.left;
                    break;
                default:
                    deltaX = (viewWidth - width) / 2 - rect.left;
                    break;
            }
            mScrollEdge = EDGE_BOTH;
        } else if (rect.left > 0) {
            mScrollEdge = EDGE_LEFT;
            deltaX = -rect.left;
        } else if (rect.right < viewWidth) {
            deltaX = viewWidth - rect.right;
            mScrollEdge = EDGE_RIGHT;
        } else {
            mScrollEdge = EDGE_NONE;
        }

        // 将x,y方向上移动的距离保存到mSuppMatrix
        mSuppMatrix.postTranslate(deltaX, deltaY);
        return true;
    }
```

8.PhotoViewAttacher的getDrawMatrix
```java
private Matrix getDrawMatrix() {
        // mBaseMatrix是在updateBaseMatrix方法中使用的，跳转10
        mDrawMatrix.set(mBaseMatrix);
        // mBaseMatrix和mSuppMatrix合并
        mDrawMatrix.postConcat(mSuppMatrix);
        return mDrawMatrix;
    }
```

9.PhotoViewAttacher的setImageViewMatrix，此处执行完一次onDrag完成
```java
private void setImageViewMatrix(Matrix matrix) {
        // 调用ImageView的setImageMatrix将矩阵应用到imageView的drawable上
        mImageView.setImageMatrix(matrix);

        // Call MatrixChangedListener if needed
        if (mMatrixChangeListener != null) {
            RectF displayRect = getDisplayRect(matrix);
            if (displayRect != null) {
                mMatrixChangeListener.onMatrixChanged(displayRect);
            }
        }
    }
```

10.PhotoViewAttacher的updateBaseMatrix
```java
private void updateBaseMatrix(Drawable drawable) {
        if (drawable == null) {
            return;
        }

        // 获取imageView和drawable的宽高
        final float viewWidth = getImageViewWidth(mImageView);
        final float viewHeight = getImageViewHeight(mImageView);
        final int drawableWidth = drawable.getIntrinsicWidth();
        final int drawableHeight = drawable.getIntrinsicHeight();

        // 讲mBaseMatrix重置为单位矩阵
        mBaseMatrix.reset();

        final float widthScale = viewWidth / drawableWidth;
        final float heightScale = viewHeight / drawableHeight;

        // 根据不同的ScaleType计算相应的矩阵，计算结果保存在mBaseMatrix中，省略
        // ...

        resetMatrix();
    }
```

11.PhotoViewAttacher实现OnGestureListener的onFling
```java
    @Override
    public void onFling(float startX, float startY, float velocityX,
                        float velocityY) {
        // 此处向主线程post了一个FlingRunnable，FlingRunnable是PhotoViewAttacher的内部类，跳转12
        mCurrentFlingRunnable = new FlingRunnable(mImageView.getContext());
        mCurrentFlingRunnable.fling(getImageViewWidth(mImageView),
                getImageViewHeight(mImageView), (int) velocityX, (int) velocityY);
        mImageView.post(mCurrentFlingRunnable);
    }
```

12.FlingRunnable
```java
private class FlingRunnable implements Runnable {

        private final OverScroller mScroller;
        private int mCurrentX, mCurrentY;

        public FlingRunnable(Context context) {
            mScroller = new OverScroller(context);
        }

        public void cancelFling() {
            mScroller.forceFinished(true);
        }

        // fling的条件是当前iamgeView的width或height小于getDisplayRect获得的rect的width或height，简单说就是图片超出了imageView的范围
        public void fling(int viewWidth, int viewHeight, int velocityX,
                          int velocityY) {
            final RectF rect = getDisplayRect();
            if (rect == null) {
                return;
            }

            // startX为Math.round(-rect.left)，注意理解负号，首先如果要fling，上面说过了图片超出了imageView的范围，所以 -rect.left实际上是一个正的值！这个值在0到rect.width() - viewWidth范围之间
            final int startX = Math.round(-rect.left);
            final int minX, maxX, minY, maxY;

            if (viewWidth < rect.width()) {
                minX = 0;
                maxX = Math.round(rect.width() - viewWidth);
            } else {
                minX = maxX = startX;
            }

            // 同见startX的解释
            final int startY = Math.round(-rect.top);
            if (viewHeight < rect.height()) {
                minY = 0;
                maxY = Math.round(rect.height() - viewHeight);
            } else {
                minY = maxY = startY;
            }

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

