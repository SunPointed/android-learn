# SubsamplingScaleImageView 学习
A custom image view for Android, designed for photo galleries and displaying huge images (e.g. maps and building plans) without `OutOfMemoryError`s. Includes pinch to zoom, panning, rotation and animation support, and allows easy extension so you can add your own overlays and touch event detection.

* [项目地址](https://github.com/davemorrissey/subsampling-scale-image-view)

# 要点
SubsamplingScaleImageView主要逻辑：
1.setImage方法，该方法会触发图片的加载。先reset所有属性，通过TilesInitTask异步处理后，获取了原图片的宽高信息；接着在主线程中回调onTilesInited方法，该方法把原图片片宽高信息保存在SubsamplingScaleImageView实例的属性中；再调用initialiseBaseLayer方法，根据fullImageSampleSize（使用瓦片的个数，瓦片就是把一张大的图片分成若干小的矩形去加载，也是SubsamplingScaleImageView的实现方式）做出不同处理；调用initialiseTileMap方法初始化tileMap（Map<Integer, List<Tile>>，实际是LinkedHashMap，其中key为从fullImageSampleSize一直到1的sampleSize，且sampleSize为2的指数，List<Tile>为该sampleSize对应的tile信息）后，通过TileLoadTask异步加载当前fullImageSampleSize的所有tile对应的图片，加载完毕后在主线程回调onTileLoaded刷新视图；最后调用refreshRequiredTiles（refreshRequiredTiles会将tileMap中不是当前fullImageSampleSize的key对应的List<Tile>中的tile的所有图片recycle，由于用户的触摸可能导致fullImageSampleSize的不断变化，SubsamplingScaleImageView的展示就是不断异步加载图片，不断recycle图片的过程），刷新tileMap中的数据。对应2-7-8-9-10。
2.onDraw方法，负责在图片加载完毕后绘图。对应5
3.onTouchEvent方法，负责事件的分发。对应3-4
4.其他还涉及矩阵的操作，BitmapFactory的使用，Uri的使用，用handler的postDelay是先长按事件等等


## 1.构造函数
```java
public SubsamplingScaleImageView(Context context, AttributeSet attr) {
        super(context, attr);
        density = getResources().getDisplayMetrics().density;
        // 设置原图像像素最大的放大
        setMinimumDpi(160);
        // 设置双击原图像像素最大的放大
        setDoubleTapZoomDpi(160);
        // 设置手势监听
        setGestureDetector(context);
        //构造处理长按事件的Handler
        this.handler = new Handler(new Handler.Callback() {
            public boolean handleMessage(Message message) {
                if (message.what == MESSAGE_LONG_CLICK && onLongClickListener != null) {
                    // maxTouchCount重置为0
                    maxTouchCount = 0;
                    SubsamplingScaleImageView.super.setOnLongClickListener(onLongClickListener);
                    // 调用onLongClickListener的onLongClick
                    performLongClick();
                    SubsamplingScaleImageView.super.setOnLongClickListener(null);
                }
                return true;
            }
        });
        // ... TypedArray处理xml中的属性

        // 初始化quickScaleThreshold
        quickScaleThreshold = TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_DIP, 20, context.getResources().getDisplayMetrics());
    }
```

## 2.setImage，设置view展示图片的来源
```java
    /**
     * Set the image source from a bitmap, resource, asset, file or other URI, providing a preview image to be
     * displayed until the full size image is loaded, starting with a given orientation setting, scale and center.
     * This is the best method to use when you want scale and center to be restored after screen orientation change;
     * it avoids any redundant loading of tiles in the wrong orientation.
     *
     * You must declare the dimensions of the full size image by calling {@link ImageSource#dimensions(int, int)}
     * on the imageSource object. The preview source will be ignored if you don't provide dimensions,
     * and if you provide a bitmap for the full size image.
     * @param imageSource Image source. Dimensions must be declared.
     * @param previewSource Optional source for a preview image to be displayed and allow interaction while the full size image loads.
     * @param state State to be restored. Nullable.
     */
    public final void setImage(ImageSource imageSource, ImageSource previewSource, ImageViewState state) {
        // ...

        // 重置所有属性
        reset(true);
        // 根据已有的state设置图片的缩放、中心和方向
        if (state != null) { restoreState(state); }

        // 根据previewSource异步加载图片
        if (previewSource != null) {
            // ...
            this.sWidth = imageSource.getSWidth();
            this.sHeight = imageSource.getSHeight();
            this.pRegion = previewSource.getSRegion();
            if (previewSource.getBitmap() != null) {
                this.bitmapIsCached = previewSource.isCached();
                onPreviewLoaded(previewSource.getBitmap());
            } else {
                Uri uri = previewSource.getUri();
                if (uri == null && previewSource.getResource() != null) {
                    uri = Uri.parse(ContentResolver.SCHEME_ANDROID_RESOURCE + "://" + getContext().getPackageName() + "/" + previewSource.getResource());
                }
                BitmapLoadTask task = new BitmapLoadTask(this, getContext(), bitmapDecoderFactory, uri, true);
                execute(task);
            }
        }

        if (imageSource.getBitmap() != null && imageSource.getSRegion() != null) {
            // 将this.bitmap以及this的一些属性重新赋值，checkReady和checkImageLoaded后，刷新界面
            onImageLoaded(Bitmap.createBitmap(imageSource.getBitmap(), imageSource.getSRegion().left, imageSource.getSRegion().top, imageSource.getSRegion().width(), imageSource.getSRegion().height()), ORIENTATION_0, false);
        } else if (imageSource.getBitmap() != null) {
            // 将this.bitmap以及this的一些属性重新赋值，checkReady和checkImageLoaded后，刷新界面
            onImageLoaded(imageSource.getBitmap(), ORIENTATION_0, imageSource.isCached());
        } else {
            sRegion = imageSource.getSRegion();
            uri = imageSource.getUri();
            if (uri == null && imageSource.getResource() != null) {
                uri = Uri.parse(ContentResolver.SCHEME_ANDROID_RESOURCE + "://" + getContext().getPackageName() + "/" + imageSource.getResource());
            }
            
            if (imageSource.getTile() || sRegion != null) {
                // Load the bitmap using tile decoding.
                 // setImage(ImageSource imageSource)都会执行到这一步，都会采取瓦片的加载方式，最后会调用到onTilesInited
                TilesInitTask task = new TilesInitTask(this, getContext(), regionDecoderFactory, uri);
                execute(task);
            } else {
                // Load the bitmap as a single image.
                // 不使用瓦片的加载方式，加载一张完整图片
                BitmapLoadTask task = new BitmapLoadTask(this, getContext(), bitmapDecoderFactory, uri, false);
                execute(task);
            }
        }
    }
```

## 3.onTouchEvent，处理触摸事件
```java
    /**
     * Handle touch events. One finger pans, and two finger pinch and zoom plus panning.
     */
    @Override
    public boolean onTouchEvent(@NonNull MotionEvent event) {
        // During non-interruptible anims, ignore all touch events
        // ...

        // Abort if not ready
        // ...

        // Detect flings, taps and double taps
        // detector.onTouchEven，GestureDetector对手势的处理，GestureDetector消耗了事件则返回
        if (!isQuickScaling && (detector == null || detector.onTouchEvent(event))) {
            isZooming = false;
            isPanning = false;
            maxTouchCount = 0;
            return true;
        }

        if (vTranslateStart == null) { vTranslateStart = new PointF(0, 0); }
        if (vTranslateBefore == null) { vTranslateBefore = new PointF(0, 0); }
        if (vCenterStart == null) { vCenterStart = new PointF(0, 0); }

        // Store current values so we can send an event if they change
        float scaleBefore = scale;
        vTranslateBefore.set(vTranslate);

        // GestureDetector没有消费事件，调用onTouchEventInternal方法处理
        boolean handled = onTouchEventInternal(event);
        // sendStateChanged调用用户设置的相关接口
        sendStateChanged(scaleBefore, vTranslateBefore, ORIGIN_TOUCH);
        return handled || super.onTouchEvent(event);
    }
```

## 4.onTouchEventInternal，
```java
    private boolean onTouchEventInternal(@NonNull MotionEvent event) {
        int touchCount = event.getPointerCount();
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
            case MotionEvent.ACTION_POINTER_1_DOWN:
            case MotionEvent.ACTION_POINTER_2_DOWN:
                anim = null;
                requestDisallowInterceptTouchEvent(true);
                maxTouchCount = Math.max(maxTouchCount, touchCount);
                // 大于两个touchCount时判断是否缩放并取消长按事件
                if (touchCount >= 2) {
                    if (zoomEnabled) {
                        // Start pinch to zoom. Calculate distance between touch points and center point of the pinch.
                        // distance方法计算两个点之间的距离
                        float distance = distance(event.getX(0), event.getX(1), event.getY(0), event.getY(1));
                        scaleStart = scale;
                        vDistStart = distance;
                        vTranslateStart.set(vTranslate.x, vTranslate.y);
                        vCenterStart.set((event.getX(0) + event.getX(1))/2, (event.getY(0) + event.getY(1))/2);
                    } else {
                        // Abort all gestures on second touch
                        maxTouchCount = 0;
                    }
                    // Cancel long click timer
                    // 拖动或缩放时取消长按事件
                    handler.removeMessages(MESSAGE_LONG_CLICK);
                } else if (!isQuickScaling) {
                    // Start one-finger pan
                    vTranslateStart.set(vTranslate.x, vTranslate.y);
                    vCenterStart.set(event.getX(), event.getY());

                    // Start long click timer
                    handler.sendEmptyMessageDelayed(MESSAGE_LONG_CLICK, 600);
                }
                return true;
            case MotionEvent.ACTION_MOVE:
                boolean consumed = false;
                if (maxTouchCount > 0) {
                    if (touchCount >= 2) {
                        // Calculate new distance between touch points, to scale and pan relative to start values.
                        float vDistEnd = distance(event.getX(0), event.getX(1), event.getY(0), event.getY(1));
                        float vCenterEndX = (event.getX(0) + event.getX(1))/2;
                        float vCenterEndY = (event.getY(0) + event.getY(1))/2;

                        if (zoomEnabled && (distance(vCenterStart.x, vCenterEndX, vCenterStart.y, vCenterEndY) > 5 || Math.abs(vDistEnd - vDistStart) > 5 || isPanning)) {
                            isZooming = true;
                            isPanning = true;
                            consumed = true;

                            // 根据distance计算缩放比例
                            double previousScale = scale;
                            scale = Math.min(maxScale, (vDistEnd / vDistStart) * scaleStart);

                            if (scale <= minScale()) {
                                // Minimum scale reached so don't pan. Adjust start settings so any expand will zoom in.
                                vDistStart = vDistEnd;
                                scaleStart = minScale();
                                vCenterStart.set(vCenterEndX, vCenterEndY);
                                vTranslateStart.set(vTranslate);
                            } else if (panEnabled) {
                                // Translate to place the source image coordinate that was at the center of the pinch at the start
                                // at the center of the pinch now, to give simultaneous pan + zoom.
                                float vLeftStart = vCenterStart.x - vTranslateStart.x;
                                float vTopStart = vCenterStart.y - vTranslateStart.y;
                                float vLeftNow = vLeftStart * (scale/scaleStart);
                                float vTopNow = vTopStart * (scale/scaleStart);
                                vTranslate.x = vCenterEndX - vLeftNow;
                                vTranslate.y = vCenterEndY - vTopNow;
                                if ((previousScale * sHeight() < getHeight() && scale * sHeight() >= getHeight()) || (previousScale * sWidth() < getWidth() && scale * sWidth() >= getWidth())) {
                                    fitToBounds(true);
                                    vCenterStart.set(vCenterEndX, vCenterEndY);
                                    vTranslateStart.set(vTranslate);
                                    scaleStart = scale;
                                    vDistStart = vDistEnd;
                                }
                            } else if (sRequestedCenter != null) {
                                // With a center specified from code, zoom around that point.
                                vTranslate.x = (getWidth()/2) - (scale * sRequestedCenter.x);
                                vTranslate.y = (getHeight()/2) - (scale * sRequestedCenter.y);
                            } else {
                                // With no requested center, scale around the image center.
                                vTranslate.x = (getWidth()/2) - (scale * (sWidth()/2));
                                vTranslate.y = (getHeight()/2) - (scale * (sHeight()/2));
                            }

                            fitToBounds(true);
                            refreshRequiredTiles(false);
                        }
                    } else if (isQuickScaling) {
                        // One finger zoom
                        // Stole Google's Magical Formula™ to make sure it feels the exact same
                        float dist = Math.abs(quickScaleVStart.y - event.getY()) * 2 + quickScaleThreshold;

                        if (quickScaleLastDistance == -1f) {
                            quickScaleLastDistance = dist;
                        }
                        boolean isUpwards = event.getY() > quickScaleVLastPoint.y;
                        quickScaleVLastPoint.set(0, event.getY());

                        float spanDiff = Math.abs(1 - (dist / quickScaleLastDistance)) * 0.5f;

                        if (spanDiff > 0.03f || quickScaleMoved) {
                            quickScaleMoved = true;

                            float multiplier = 1;
                            if (quickScaleLastDistance > 0) {
                                multiplier = isUpwards ? (1 + spanDiff) : (1 - spanDiff);
                            }

                            double previousScale = scale;
                            scale = Math.max(minScale(), Math.min(maxScale, scale * multiplier));

                            if (panEnabled) {
                                float vLeftStart = vCenterStart.x - vTranslateStart.x;
                                float vTopStart = vCenterStart.y - vTranslateStart.y;
                                float vLeftNow = vLeftStart * (scale/scaleStart);
                                float vTopNow = vTopStart * (scale/scaleStart);
                                vTranslate.x = vCenterStart.x - vLeftNow;
                                vTranslate.y = vCenterStart.y - vTopNow;
                                if ((previousScale * sHeight() < getHeight() && scale * sHeight() >= getHeight()) || (previousScale * sWidth() < getWidth() && scale * sWidth() >= getWidth())) {
                                    fitToBounds(true);
                                    vCenterStart.set(sourceToViewCoord(quickScaleSCenter));
                                    vTranslateStart.set(vTranslate);
                                    scaleStart = scale;
                                    dist = 0;
                                }
                            } else if (sRequestedCenter != null) {
                                // With a center specified from code, zoom around that point.
                                vTranslate.x = (getWidth()/2) - (scale * sRequestedCenter.x);
                                vTranslate.y = (getHeight()/2) - (scale * sRequestedCenter.y);
                            } else {
                                // With no requested center, scale around the image center.
                                vTranslate.x = (getWidth()/2) - (scale * (sWidth()/2));
                                vTranslate.y = (getHeight()/2) - (scale * (sHeight()/2));
                            }
                        }

                        quickScaleLastDistance = dist;

                        fitToBounds(true);
                        refreshRequiredTiles(false);

                        consumed = true;
                    } else if (!isZooming) {
                        // One finger pan - translate the image. We do this calculation even with pan disabled so click
                        // and long click behaviour is preserved.
                        float dx = Math.abs(event.getX() - vCenterStart.x);
                        float dy = Math.abs(event.getY() - vCenterStart.y);

                        //On the Samsung S6 long click event does not work, because the dx > 5 usually true
                        float offset = density * 5;
                        if (dx > offset || dy > offset || isPanning) {
                            consumed = true;
                            vTranslate.x = vTranslateStart.x + (event.getX() - vCenterStart.x);
                            vTranslate.y = vTranslateStart.y + (event.getY() - vCenterStart.y);

                            float lastX = vTranslate.x;
                            float lastY = vTranslate.y;
                            fitToBounds(true);
                            boolean atXEdge = lastX != vTranslate.x;
                            boolean atYEdge = lastY != vTranslate.y;
                            boolean edgeXSwipe = atXEdge && dx > dy && !isPanning;
                            boolean edgeYSwipe = atYEdge && dy > dx && !isPanning;
                            boolean yPan = lastY == vTranslate.y && dy > offset * 3;
                            if (!edgeXSwipe && !edgeYSwipe && (!atXEdge || !atYEdge || yPan || isPanning)) {
                                isPanning = true;
                            } else if (dx > offset || dy > offset) {
                                // Haven't panned the image, and we're at the left or right edge. Switch to page swipe.
                                maxTouchCount = 0;
                                handler.removeMessages(MESSAGE_LONG_CLICK);
                                requestDisallowInterceptTouchEvent(false);
                            } 
                            if (!panEnabled) {
                                vTranslate.x = vTranslateStart.x;
                                vTranslate.y = vTranslateStart.y;
                                requestDisallowInterceptTouchEvent(false);
                            }

                            refreshRequiredTiles(false);
                        }
                    }
                }
                if (consumed) {
                    handler.removeMessages(MESSAGE_LONG_CLICK);
                    invalidate();
                    return true;
                }
                break;
            case MotionEvent.ACTION_UP:
            case MotionEvent.ACTION_POINTER_UP:
            case MotionEvent.ACTION_POINTER_2_UP:
                handler.removeMessages(MESSAGE_LONG_CLICK);
                if (isQuickScaling) {
                    isQuickScaling = false;
                    if (!quickScaleMoved) {
                        doubleTapZoom(quickScaleSCenter, vCenterStart);
                    }
                }
                if (maxTouchCount > 0 && (isZooming || isPanning)) {
                    if (isZooming && touchCount == 2) {
                        // Convert from zoom to pan with remaining touch
                        isPanning = true;
                        vTranslateStart.set(vTranslate.x, vTranslate.y);
                        if (event.getActionIndex() == 1) {
                            vCenterStart.set(event.getX(0), event.getY(0));
                        } else {
                            vCenterStart.set(event.getX(1), event.getY(1));
                        }
                    }
                    if (touchCount < 3) {
                        // End zooming when only one touch point
                        isZooming = false;
                    }
                    if (touchCount < 2) {
                        // End panning when no touch points
                        isPanning = false;
                        maxTouchCount = 0;
                    }
                    // Trigger load of tiles now required
                    refreshRequiredTiles(true);
                    return true;
                }
                if (touchCount == 1) {
                    isZooming = false;
                    isPanning = false;
                    maxTouchCount = 0;
                }
                return true;
        }
        return false;
    }
```

## 5.onDraw
```java
    /**
     * Draw method should not be called until the view has dimensions so the first calls are used as triggers to calculate
     * the scaling and tiling required. Once the view is setup, tiles are displayed as they are loaded.
     */
    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        createPaints();

        // If image or view dimensions are not known yet, abort.
        if (sWidth == 0 || sHeight == 0 || getWidth() == 0 || getHeight() == 0) {
            return;
        }

        // When using tiles, on first render with no tile map ready, initialise it and kick off async base image loading.
        if (tileMap == null && decoder != null) {
            initialiseBaseLayer(getMaxBitmapDimensions(canvas));
        }

        // If image has been loaded or supplied as a bitmap, onDraw may be the first time the view has
        // dimensions and therefore the first opportunity to set scale and translate. If this call returns
        // false there is nothing to be drawn so return immediately.
        if (!checkReady()) {
            return;
        }

        // Set scale and translate before draw.
        preDraw();

        // If animating scale, calculate current scale and center with easing equations
        if (anim != null) {
            // Store current values so we can send an event if they change
            float scaleBefore = scale;
            if (vTranslateBefore == null) { vTranslateBefore = new PointF(0, 0); }
            vTranslateBefore.set(vTranslate);

            // scaleElapsed大于duration说明动画已经完成
            long scaleElapsed = System.currentTimeMillis() - anim.time;
            boolean finished = scaleElapsed > anim.duration;
            scaleElapsed = Math.min(scaleElapsed, anim.duration);
            // ease根据scaleElapsed计算scale
            scale = ease(anim.easing, scaleElapsed, anim.scaleStart, anim.scaleEnd - anim.scaleStart, anim.duration);

            // Apply required animation to the focal point
            // 将动画的效果作用到焦点上
            float vFocusNowX = ease(anim.easing, scaleElapsed, anim.vFocusStart.x, anim.vFocusEnd.x - anim.vFocusStart.x, anim.duration);
            float vFocusNowY = ease(anim.easing, scaleElapsed, anim.vFocusStart.y, anim.vFocusEnd.y - anim.vFocusStart.y, anim.duration);
            // Find out where the focal point is at this scale and adjust its position to follow the animation path
            vTranslate.x -= sourceToViewX(anim.sCenterEnd.x) - vFocusNowX;
            vTranslate.y -= sourceToViewY(anim.sCenterEnd.y) - vFocusNowY;

            // For translate anims, showing the image non-centered is never allowed, for scaling anims it is during the animation.
            // fitToBounds方法最主要的作用是计算scale变量和vTranslate变量
            fitToBounds(finished || (anim.scaleStart == anim.scaleEnd));
            // 调用用户设置的方法
            sendStateChanged(scaleBefore, vTranslateBefore, anim.origin);

            refreshRequiredTiles(finished);
            
            if (finished) {
                if (anim.listener != null) {
                    try {
                        anim.listener.onComplete();
                    } catch (Exception e) {
                        Log.w(TAG, "Error thrown by animation listener", e);
                    }
                }
                // finished为true说明该动画完成置为null
                anim = null;
            }
            // When you call invalidate() you can assume that the onDraw(
            // canvas c) is going to be called, yes.But be careful with 
            // calling the nextFrame inside the onDraw(), be sure to control 
            // the refreshing time.
            invalidate();
        }

        if (tileMap != null && isBaseLayerReady()) {

            // Optimum sample size for current scale
            int sampleSize = Math.min(fullImageSampleSize, calculateInSampleSize(scale));

            // First check for missing tiles - if there are any we need the base layer underneath to avoid gaps
            boolean hasMissingTiles = false;
            for (Map.Entry<Integer, List<Tile>> tileMapEntry : tileMap.entrySet()) {
                if (tileMapEntry.getKey() == sampleSize) {
                    for (Tile tile : tileMapEntry.getValue()) {
                        if (tile.visible && (tile.loading || tile.bitmap == null)) {
                            hasMissingTiles = true;
                        }
                    }
                }
            }

            // Render all loaded tiles. LinkedHashMap used for bottom up rendering - lower res tiles underneath.
            for (Map.Entry<Integer, List<Tile>> tileMapEntry : tileMap.entrySet()) {
                if (tileMapEntry.getKey() == sampleSize || hasMissingTiles) {
                    for (Tile tile : tileMapEntry.getValue()) {
                        // 图片的矩形转换为View的矩形
                        sourceToViewRect(tile.sRect, tile.vRect);
                        if (!tile.loading && tile.bitmap != null) {
                            // 画背景
                            if (tileBgPaint != null) {
                                canvas.drawRect(tile.vRect, tileBgPaint);
                            }
                            if (matrix == null) { matrix = new Matrix(); }
                            matrix.reset();
                            // 设置srcArray和dstArray
                            setMatrixArray(srcArray, 0, 0, tile.bitmap.getWidth(), 0, tile.bitmap.getWidth(), tile.bitmap.getHeight(), 0, tile.bitmap.getHeight());
                            if (getRequiredRotation() == ORIENTATION_0) {
                                setMatrixArray(dstArray, tile.vRect.left, tile.vRect.top, tile.vRect.right, tile.vRect.top, tile.vRect.right, tile.vRect.bottom, tile.vRect.left, tile.vRect.bottom);
                            } else if (getRequiredRotation() == ORIENTATION_90) {
                                setMatrixArray(dstArray, tile.vRect.right, tile.vRect.top, tile.vRect.right, tile.vRect.bottom, tile.vRect.left, tile.vRect.bottom, tile.vRect.left, tile.vRect.top);
                            } else if (getRequiredRotation() == ORIENTATION_180) {
                                setMatrixArray(dstArray, tile.vRect.right, tile.vRect.bottom, tile.vRect.left, tile.vRect.bottom, tile.vRect.left, tile.vRect.top, tile.vRect.right, tile.vRect.top);
                            } else if (getRequiredRotation() == ORIENTATION_270) {
                                setMatrixArray(dstArray, tile.vRect.left, tile.vRect.bottom, tile.vRect.left, tile.vRect.top, tile.vRect.right, tile.vRect.top, tile.vRect.right, tile.vRect.bottom);
                            }
                            // setPolyToPoly方法将srcArray中的点变换到dstArray点所需的矩阵运算保存在matrix中
                            matrix.setPolyToPoly(srcArray, 0, dstArray, 0, 4);
                            // 将matrix作用到图片上绘图
                            canvas.drawBitmap(tile.bitmap, matrix, bitmapPaint);
                            // ...debug
                        } 
                        // ...debug
                    }
                }
            }

        } 
        // tileMap为null时，直接画bitmap
        else if (bitmap != null) {

            float xScale = scale, yScale = scale;
            if (bitmapIsPreview) {
                xScale = scale * ((float)sWidth/bitmap.getWidth());
                yScale = scale * ((float)sHeight/bitmap.getHeight());
            }

            if (matrix == null) { matrix = new Matrix(); }
            matrix.reset();
            // 缩放，旋转，位移
            matrix.postScale(xScale, yScale);
            matrix.postRotate(getRequiredRotation());
            matrix.postTranslate(vTranslate.x, vTranslate.y);

            // 根据旋转方向的不同进行旋转后的位移调整
            if (getRequiredRotation() == ORIENTATION_180) {
                matrix.postTranslate(scale * sWidth, scale * sHeight);
            } else if (getRequiredRotation() == ORIENTATION_90) {
                matrix.postTranslate(scale * sHeight, 0);
            } else if (getRequiredRotation() == ORIENTATION_270) {
                matrix.postTranslate(0, scale * sWidth);
            }

            // 画背景
            if (tileBgPaint != null) {
                if (sRect == null) { sRect = new RectF(); }
                sRect.set(0f, 0f, bitmapIsPreview ? bitmap.getWidth() : sWidth, bitmapIsPreview ? bitmap.getHeight() : sHeight);
                matrix.mapRect(sRect);
                canvas.drawRect(sRect, tileBgPaint);
            }
            // 画图
            canvas.drawBitmap(bitmap, matrix, bitmapPaint);

        }

        // ...debug
    }
```

## 6.refreshRequiredTiles
```java
    /**
     * Loads the optimum tiles for display at the current scale and translate, so the screen can be filled with tiles
     * that are at least as high resolution as the screen. Frees up bitmaps that are now off the screen.
     * @param load Whether to load the new tiles needed. Use false while scrolling/panning for performance.
     */
    private void refreshRequiredTiles(boolean load) {
        if (decoder == null || tileMap == null) { return; }

        int sampleSize = Math.min(fullImageSampleSize, calculateInSampleSize(scale));

        // Load tiles of the correct sample size that are on screen. Discard tiles off screen, and those that are higher
        // resolution than required, or lower res than required but not the base layer, so the base layer is always present.
        for (Map.Entry<Integer, List<Tile>> tileMapEntry : tileMap.entrySet()) {
            for (Tile tile : tileMapEntry.getValue()) {
                // 把不是当前sampleSize的tile的bitmap回收，防止内存溢出
                if (tile.sampleSize < sampleSize || (tile.sampleSize > sampleSize && tile.sampleSize != fullImageSampleSize)) {
                    tile.visible = false;
                    if (tile.bitmap != null) {
                        tile.bitmap.recycle();
                        tile.bitmap = null;
                    }
                }
                // 是当前的sampleSize的tile
                if (tile.sampleSize == sampleSize) {
                    // 判断tile是否可见，可见才加载
                    if (tileVisible(tile)) {
                        tile.visible = true;
                        if (!tile.loading && tile.bitmap == null && load) {
                            // 用AsyncTask异步加载tile的bitmap
                            TileLoadTask task = new TileLoadTask(this, decoder, tile);
                            execute(task);
                        }
                    } 
                    // 对于不可见的tile，如果sampleSize不是fullImageSampleSize，则回收，猜测fullImageSampleSize用于显示完整图片
                    else if (tile.sampleSize != fullImageSampleSize) {
                        tile.visible = false;
                        if (tile.bitmap != null) {
                            tile.bitmap.recycle();
                            tile.bitmap = null;
                        }
                    }
                } else if (tile.sampleSize == fullImageSampleSize) {
                    tile.visible = true;
                }
            }
        }

    }
```

## 7.onTilesInited
```java
    /**
     * Called by worker task when decoder is ready and image size and EXIF orientation is known.
     */
    private synchronized void onTilesInited(ImageRegionDecoder decoder, int sWidth, int sHeight, int sOrientation) {
        debug("onTilesInited sWidth=%d, sHeight=%d, sOrientation=%d", sWidth, sHeight, orientation);
        // If actual dimensions don't match the declared size, reset everything.
        if (this.sWidth > 0 && this.sHeight > 0 && (this.sWidth != sWidth || this.sHeight != sHeight)) {
            reset(false);
            if (bitmap != null) {
                if (!bitmapIsCached) {
                    bitmap.recycle();
                }
                bitmap = null;
                if (onImageEventListener != null && bitmapIsCached) {
                    onImageEventListener.onPreviewReleased();
                }
                bitmapIsPreview = false;
                bitmapIsCached = false;
            }
        }
        this.decoder = decoder;
        this.sWidth = sWidth;
        this.sHeight = sHeight;
        this.sOrientation = sOrientation;
        checkReady();
        if (!checkImageLoaded() && maxTileWidth > 0 && maxTileWidth != TILE_SIZE_AUTO && maxTileHeight > 0 && maxTileHeight != TILE_SIZE_AUTO && getWidth() > 0 && getHeight() > 0) {
            initialiseBaseLayer(new Point(maxTileWidth, maxTileHeight));
        }
        invalidate();
        requestLayout();
    }
```

## 8.initialiseBaseLayer
```java
    /**
     * Called on first draw when the view has dimensions. Calculates the initial sample size and starts async loading of
     * the base layer image - the whole source subsampled as necessary.
     */
    private synchronized void initialiseBaseLayer(Point maxTileDimensions) {
        debug("initialiseBaseLayer maxTileDimensions=%dx%d", maxTileDimensions.x, maxTileDimensions.y);

        satTemp = new ScaleAndTranslate(0f, new PointF(0, 0));
        fitToBounds(true, satTemp);

        // Load double resolution - next level will be split into four tiles and at the center all four are required,
        // so don't bother with tiling until the next level 16 tiles are needed.
        // fitToBounds后根据原图片长宽与View长宽的比例，satTemp.scale是一个小于等于的值2
        // fullImageSampleSize是当前使用瓦片的个数
        fullImageSampleSize = calculateInSampleSize(satTemp.scale);
        if (fullImageSampleSize > 1) {
            fullImageSampleSize /= 2;
        }

        if (fullImageSampleSize == 1 && sRegion == null && sWidth() < maxTileDimensions.x && sHeight() < maxTileDimensions.y) {

            // Whole image is required at native resolution, and is smaller than the canvas max bitmap size.
            // Use BitmapDecoder for better image support.
            decoder.recycle();
            decoder = null;
            BitmapLoadTask task = new BitmapLoadTask(this, getContext(), bitmapDecoderFactory, uri, false);
            execute(task);

        } else {

            initialiseTileMap(maxTileDimensions);

            List<Tile> baseGrid = tileMap.get(fullImageSampleSize);
            for (Tile baseTile : baseGrid) {
                TileLoadTask task = new TileLoadTask(this, decoder, baseTile);
                execute(task);
            }
            refreshRequiredTiles(true);

        }

    }
```

## 9.initialiseTileMap
```java
    /**
     * Once source image and view dimensions are known, creates a map of sample size to tile grid.
     */
    private void initialiseTileMap(Point maxTileDimensions) {
        debug("initialiseTileMap maxTileDimensions=%dx%d", maxTileDimensions.x, maxTileDimensions.y);
        this.tileMap = new LinkedHashMap<>();
        int sampleSize = fullImageSampleSize;
        int xTiles = 1;
        int yTiles = 1;
        // 把相应数据填入tileMap，知道sampleSize为1时结束
        while (true) {
            int sTileWidth = sWidth()/xTiles;
            int sTileHeight = sHeight()/yTiles;
            int subTileWidth = sTileWidth/sampleSize;
            int subTileHeight = sTileHeight/sampleSize;
            while (subTileWidth + xTiles + 1 > maxTileDimensions.x || (subTileWidth > getWidth() * 1.25 && sampleSize < fullImageSampleSize)) {
                xTiles += 1;
                sTileWidth = sWidth()/xTiles;
                subTileWidth = sTileWidth/sampleSize;
            }
            while (subTileHeight + yTiles + 1 > maxTileDimensions.y || (subTileHeight > getHeight() * 1.25 && sampleSize < fullImageSampleSize)) {
                yTiles += 1;
                sTileHeight = sHeight()/yTiles;
                subTileHeight = sTileHeight/sampleSize;
            }
            List<Tile> tileGrid = new ArrayList<>(xTiles * yTiles);
            for (int x = 0; x < xTiles; x++) {
                for (int y = 0; y < yTiles; y++) {
                    Tile tile = new Tile();
                    tile.sampleSize = sampleSize;
                    tile.visible = sampleSize == fullImageSampleSize;
                    tile.sRect = new Rect(
                        x * sTileWidth,
                        y * sTileHeight,
                        x == xTiles - 1 ? sWidth() : (x + 1) * sTileWidth,
                        y == yTiles - 1 ? sHeight() : (y + 1) * sTileHeight
                    );
                    tile.vRect = new Rect(0, 0, 0, 0);
                    tile.fileSRect = new Rect(tile.sRect);
                    tileGrid.add(tile);
                }
            }
            tileMap.put(sampleSize, tileGrid);
            if (sampleSize == 1) {
                break;
            } else {
                sampleSize /= 2;
            }
        }
    }
```

## 10.onTileLoaded
```java
    /**
     * Called by worker task when a tile has loaded. Redraws the view.
     */
    private synchronized void onTileLoaded() {
        debug("onTileLoaded");
        checkReady();
        checkImageLoaded();
        if (isBaseLayerReady() && bitmap != null) {
            if (!bitmapIsCached) {
                bitmap.recycle();
            }
            bitmap = null;
            if (onImageEventListener != null && bitmapIsCached) {
                onImageEventListener.onPreviewReleased();
            }
            bitmapIsPreview = false;
            bitmapIsCached = false;
        }
        invalidate();
    }
```