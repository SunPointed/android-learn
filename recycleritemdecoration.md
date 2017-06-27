# RecyclerItemDecoration 学习
RecyclerItemDecoration allows you to draw divider between items in recyclerview with multiple ViewType without considering items' positions!

When using recyclerView with different ViewType, you either have only one simple divider or different types of dividers. When you want to draw different dividers between recyclerView's items, basically you must consider items' position; often you need to have separate ItemDecoration's behaviors declared in your code using switch cases or if statements. For example, each time items' position changes happen, you must rewrite ItemDecoration's behaviors.

You don't need to think about items' position! You need to care about their ViewType!!

* [项目地址](https://github.com/magiepooh/RecyclerItemDecoration)

## 使用
```java
    RecyclerView.ItemDecoration decoration = ItemDecorations.vertical(this)
                .first(R.drawable.shape_decoration_green_h_16)
                .type(DemoViewType.LANDSCAPE_TILE.ordinal(), R.drawable.shape_decoration_cornflower_lilac_h_8)
                .type(DemoViewType.LANDSCAPE_ITEM.ordinal(), R.drawable.shape_decoration_gray_h_12_padding)
                .type(DemoViewType.LANDSCAPE_DESCRIPTION.ordinal(), R.drawable.shape_decoration_red_h_8)
                .last(R.drawable.shape_decoration_flush_orange_h_16)
                .create();

    recyclerView.addItemDecoration(decoration);
```

## RecyclerView.ItemDecoration

# getItemOffsets
getItemOffsets中为outRect设置的4个方向的值，将被计算进所有 decoration 的尺寸中，而这个尺寸，被计入了 RecyclerView 每个 item view 的 padding 中

# onDraw
Draw any appropriate decorations into the Canvas supplied to the RecyclerView. Any content drawn by this method will be drawn before the item views are drawn, and will thus appear underneath the views

# onDrawOver
Draw any appropriate decorations into the Canvas supplied to the RecyclerView. Any content drawn by this method will be drawn after the item views are drawn and will thus appear over the views

## 比较简单直接贴代码
```java
    public class VerticalItemDecoration extends RecyclerView.ItemDecoration {

    private static final int[] ATTRS = {android.R.attr.listDivider};

    // 该map用来保存不同type需要使用的不同drawable
    private final Map<Integer, Drawable> mDividerViewTypeMap;
    private final Drawable mFirstDrawable;
    private final Drawable mLastDrawable;

    public VerticalItemDecoration(Map<Integer, Drawable> dividerViewTypeMap,
            Drawable firstDrawable, Drawable lastDrawable) {
        mDividerViewTypeMap = dividerViewTypeMap;
        mFirstDrawable = firstDrawable;
        mLastDrawable = lastDrawable;
    }

    @Override
    public void getItemOffsets(Rect outRect, View view, RecyclerView parent,
            RecyclerView.State state) {

        // last position
        if (isLastPosition(view, parent)) {
            if (mLastDrawable != null) {
                outRect.bottom = mLastDrawable.getIntrinsicHeight();
            }
            return;
        }

        // specific view type
        int childType = parent.getLayoutManager().getItemViewType(view);
        Drawable drawable = mDividerViewTypeMap.get(childType);
        if (drawable != null) {
            outRect.bottom = drawable.getIntrinsicHeight();
        }

        // first position
        if (isFirstPosition(view, parent) && mFirstDrawable != null) {
            outRect.top = mFirstDrawable.getIntrinsicHeight();
        }
    }

    @Override
    public void onDraw(Canvas c, RecyclerView parent, RecyclerView.State state) {
        int left = parent.getPaddingLeft();
        int right = parent.getWidth() - parent.getPaddingRight();

        int childCount = parent.getChildCount();
        for (int i = 0; i <= childCount - 1; i++) {
            View child = parent.getChildAt(i);
            int childViewType = parent.getLayoutManager().getItemViewType(child);
            RecyclerView.LayoutParams params = (RecyclerView.LayoutParams) child.getLayoutParams();

            // last position
            if (isLastPosition(child, parent)) {
                if (mLastDrawable != null) {
                    int top = child.getBottom() + params.bottomMargin;
                    int bottom = top + mLastDrawable.getIntrinsicHeight();
                    mLastDrawable.setBounds(left, top, right, bottom);
                    mLastDrawable.draw(c);
                }
                return;
            }

            // specific view type
            Drawable drawable = mDividerViewTypeMap.get(childViewType);
            if (drawable != null) {
                int top = child.getBottom() + params.bottomMargin;
                int bottom = top + drawable.getIntrinsicHeight();
                drawable.setBounds(left, top, right, bottom);
                drawable.draw(c);
            }

            // first position
            if (isFirstPosition(child, parent) && mFirstDrawable != null) {
                int bottom = child.getTop() - params.topMargin;
                int top = bottom - mFirstDrawable.getIntrinsicHeight();
                mFirstDrawable.setBounds(left, top, right, bottom);
                mFirstDrawable.draw(c);
            }
        }
    }

    private boolean isFirstPosition(View view, RecyclerView parent) {
        return parent.getChildAdapterPosition(view) == 0;
    }

    private boolean isLastPosition(View view, RecyclerView parent) {
        return parent.getChildAdapterPosition(view) == parent.getAdapter().getItemCount() - 1;
    }

    // Builder类
}
```
