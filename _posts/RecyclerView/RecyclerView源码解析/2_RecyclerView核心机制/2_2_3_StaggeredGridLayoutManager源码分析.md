# StaggeredGridLayoutManager源码分析

StaggeredGridLayoutManager是RecyclerView提供的第三种内置布局管理器，用于实现瀑布流布局效果。它支持垂直和水平两种方向的布局，可以创建出像Pinterest这样的不规则网格效果。本文将深入分析StaggeredGridLayoutManager的源码实现。

## 1. 基本概念

StaggeredGridLayoutManager的核心概念是"跨度(Span)"，一个StaggeredGridLayoutManager包含多个跨度，每个Item占据一个跨度。与GridLayoutManager不同，StaggeredGridLayoutManager中的Item高度（垂直布局）或宽度（水平布局）可以不同，这就形成了参差不齐的瀑布流效果。

### 1.1 主要属性

```java
public class StaggeredGridLayoutManager extends RecyclerView.LayoutManager implements RecyclerView.SmoothScroller.ScrollVectorProvider {
    // 垂直布局
    public static final int VERTICAL = OrientationHelper.VERTICAL;
    // 水平布局
    public static final int HORIZONTAL = OrientationHelper.HORIZONTAL;
    
    // 布局方向，默认垂直
    private int mOrientation = VERTICAL;
    // 跨度数
    private int mSpanCount = -1;
    // 布局跨度数组
    private Span[] mSpans;
    
    // 布局锚点信息
    private AnchorInfo mAnchorInfo = new AnchorInfo();
    // 布局状态
    private final LayoutState mLayoutState = new LayoutState();
    
    // 辅助布局工具类
    private OrientationHelper mPrimaryOrientation;
    private OrientationHelper mSecondaryOrientation;
    
    // 其他属性...
}
```

### 1.2 初始化

StaggeredGridLayoutManager通过构造方法初始化跨度数和布局方向：

```java
public StaggeredGridLayoutManager(int spanCount, int orientation) {
    mSpanCount = spanCount;
    setOrientation(orientation);
    ensureGapWorker();
}

public void setOrientation(int orientation) {
    if (orientation != HORIZONTAL && orientation != VERTICAL) {
        throw new IllegalArgumentException("invalid orientation.");
    }
    assertNotInLayoutOrScroll(null);
    if (orientation == mOrientation && mPrimaryOrientation != null) {
        return;
    }
    mOrientation = orientation;
    if (mOrientation == VERTICAL) {
        mPrimaryOrientation = OrientationHelper.createVertical(this);
        mSecondaryOrientation = OrientationHelper.createHorizontal(this);
    } else {
        mPrimaryOrientation = OrientationHelper.createHorizontal(this);
        mSecondaryOrientation = OrientationHelper.createVertical(this);
    }
    requestLayout();
}
```

## 2. 布局实现

### 2.1 Span类

StaggeredGridLayoutManager内部定义了Span类来管理每个跨度：

```java
private class Span {
    // 这个跨度持有的Views
    ArrayList<View> mViews = new ArrayList<>();
    // 在主方向上的起始位置
    int mCachedStart = Integer.MIN_VALUE;
    // 在主方向上的结束位置
    int mCachedEnd = Integer.MIN_VALUE;
    // 跨度的索引
    int mIndex;
    
    // 构造方法
    Span(int index) {
        mIndex = index;
    }
    
    // 追加一个View到这个跨度
    void appendToSpan(View view) {
        LayoutParams lp = getLayoutParams(view);
        lp.mSpan = this;
        mViews.add(view);
        mCachedEnd = Integer.MIN_VALUE;
    }
    
    // 向这个跨度开头添加一个View
    void prependToSpan(View view) {
        LayoutParams lp = getLayoutParams(view);
        lp.mSpan = this;
        mViews.add(0, view);
        mCachedStart = Integer.MIN_VALUE;
    }
    
    // 其他方法...
}
```

### 2.2 布局策略

StaggeredGridLayoutManager的布局策略较为复杂，主要涉及以下几个方法：

- `onLayoutChildren`：主要布局方法
- `fill`：填充可见区域
- `layoutChunk`：布局单个View
- `findFirstVisibleItemPositions`/`findLastVisibleItemPositions`：查找可见Item

#### 2.2.1 onLayoutChildren

```java
@Override
public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
    // 保存当前布局状态
    ensureOrientationHelper();
    AnchorInfo anchorInfo = mAnchorInfo;
    anchorInfo.reset();
    
    // 计算锚点信息
    updateAnchorInfoForLayout(recycler, state, anchorInfo);
    
    // 清空当前布局
    detachAndScrapAttachedViews(recycler);
    
    // 根据锚点信息填充布局
    if (anchorInfo.mLayoutFromEnd) {
        // 从下向上布局
        updateLayoutStateToFillEnd(anchorInfo);
        fill(recycler, mLayoutState, state);
        // 从上向下布局
        updateLayoutStateToFillStart(anchorInfo);
        fill(recycler, mLayoutState, state);
    } else {
        // 从上向下布局
        updateLayoutStateToFillStart(anchorInfo);
        fill(recycler, mLayoutState, state);
        // 从下向上布局
        updateLayoutStateToFillEnd(anchorInfo);
        fill(recycler, mLayoutState, state);
    }
}
```

#### 2.2.2 fill方法

```java
private int fill(RecyclerView.Recycler recycler, LayoutState layoutState, RecyclerView.State state) {
    // 初始化起始位置
    int targetLine = layoutState.mAvailable;
    // 处理特殊情况
    if (layoutState.mScrollingOffset != LayoutState.SCROLLING_OFFSET_NaN) {
        if (layoutState.mAvailable < 0) {
            layoutState.mScrollingOffset += layoutState.mAvailable;
        }
        recycleByLayoutState(recycler, layoutState);
    }
    
    // 填充空白区域
    int remainingSpace = layoutState.mAvailable + layoutState.mExtra;
    while (remainingSpace > 0 && layoutState.hasMore(state)) {
        // 布局单个Item
        layoutChunk(recycler, state, layoutState, result);
        
        remainingSpace -= result.mConsumed;
        
        // 如果已经没有可布局的Item，退出循环
        if (!result.mFinished && layoutState.mScrollingOffset != LayoutState.SCROLLING_OFFSET_NaN) {
            layoutState.mScrollingOffset += result.mConsumed;
            if (layoutState.mAvailable < 0) {
                layoutState.mScrollingOffset += layoutState.mAvailable;
            }
            recycleByLayoutState(recycler, layoutState);
        }
    }
    
    return targetLine - layoutState.mAvailable;
}
```

#### 2.2.3 layoutChunk方法

```java
private void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state, LayoutState layoutState, LayoutChunkResult result) {
    // 获取下一个要布局的View
    View view = layoutState.next(recycler);
    if (view == null) {
        result.mFinished = true;
        return;
    }
    
    // 获取LayoutParams
    LayoutParams lp = (LayoutParams) view.getLayoutParams();
    
    // 确定此View应该放置在哪个跨度
    if (layoutState.mLayoutDirection == LayoutState.LAYOUT_START) {
        // 向上布局时，选择最大位置的跨度
        int spanIndex = getSpanIndex(lp.mSpan != null ? lp.mSpan.mIndex : -1, lp.mFullSpan);
        addView(view, 0);
    } else {
        // 向下布局时，选择最小位置的跨度
        int spanIndex = getSpanIndex(lp.mSpan != null ? lp.mSpan.mIndex : -1, lp.mFullSpan);
        addView(view);
    }
    
    // 测量View
    measureChildWithDecorationsAndMargin(view, lp);
    
    // 布局View
    if (layoutState.mLayoutDirection == LayoutState.LAYOUT_START) {
        // 向上布局
        layoutDecoratedWithMargins(
                view,
                lp.mSpan.mCachedEnd - getMeasuredWidth(view),
                lp.mSpan.mCachedEnd,
                lp.mSpan.getStartLine(),
                lp.mSpan.getStartLine() + getMeasuredHeight(view)
        );
    } else {
        // 向下布局
        layoutDecoratedWithMargins(
                view,
                lp.mSpan.getStartLine(),
                lp.mSpan.getStartLine() + getMeasuredWidth(view),
                lp.mSpan.mCachedStart,
                lp.mSpan.mCachedStart + getMeasuredHeight(view)
        );
    }
    
    // 更新结果
    result.mConsumed = getMeasuredSize(view, layoutState.mLayoutDirection);
    result.mFinished = false;
}
```

## 3. 特殊功能实现

### 3.1 全跨度(Full Span)

StaggeredGridLayoutManager支持设置某个Item占据全部跨度，通过LayoutParams的`mFullSpan`属性控制：

```java
public static class LayoutParams extends RecyclerView.LayoutParams {
    // 当前View所在的Span
    Span mSpan;
    // 是否占用全部跨度
    boolean mFullSpan;
    
    public LayoutParams(Context c, AttributeSet attrs) {
        super(c, attrs);
    }
    
    public LayoutParams(int width, int height) {
        super(width, height);
    }
    
    public LayoutParams(ViewGroup.MarginLayoutParams source) {
        super(source);
    }
    
    public LayoutParams(ViewGroup.LayoutParams source) {
        super(source);
    }
    
    public LayoutParams(RecyclerView.LayoutParams source) {
        super(source);
    }
    
    /**
     * 设置此Item是否占据全部跨度
     */
    public void setFullSpan(boolean fullSpan) {
        mFullSpan = fullSpan;
    }
    
    /**
     * 判断此Item是否占据全部跨度
     */
    public boolean isFullSpan() {
        return mFullSpan;
    }
}
```

### 3.2 Item对齐

StaggeredGridLayoutManager提供了两种对齐方式：

1. `GAP_HANDLING_NONE`：不处理间隙，可能会导致不对齐
2. `GAP_HANDLING_MOVE_ITEMS_BETWEEN_SPANS`：通过移动Items来弥补间隙

通过`setGapStrategy`方法设置：

```java
/**
 * 设置间隙处理策略
 */
public void setGapStrategy(int gapStrategy) {
    assertNotInLayoutOrScroll(null);
    if (gapStrategy == mGapStrategy) {
        return;
    }
    if (gapStrategy != GAP_HANDLING_NONE && gapStrategy != GAP_HANDLING_MOVE_ITEMS_BETWEEN_SPANS) {
        throw new IllegalArgumentException("invalid gap strategy. Must be GAP_HANDLING_NONE "
                + "or GAP_HANDLING_MOVE_ITEMS_BETWEEN_SPANS");
    }
    mGapStrategy = gapStrategy;
    requestLayout();
}
```

### 3.3 状态保存与恢复

StaggeredGridLayoutManager实现了状态保存与恢复功能，确保在配置变化时（如屏幕旋转）保持滚动位置：

```java
@Override
public Parcelable onSaveInstanceState() {
    if (mPendingSavedState != null) {
        return new SavedState(mPendingSavedState);
    }
    SavedState state = new SavedState();
    state.mReverseLayout = mReverseLayout;
    state.mAnchorLayoutFromEnd = mLastLayoutFromEnd;
    state.mLastLayoutRTL = mLastLayoutRTL;
    
    // 保存每个跨度的偏移量
    if (mLazySpanLookup != null && mLazySpanLookup.mData != null) {
        state.mSpanLookup = mLazySpanLookup.mData;
        state.mSpanLookupSize = state.mSpanLookup.length;
        state.mFullSpanItems = mLazySpanLookup.mFullSpanItems;
    } else {
        state.mSpanLookupSize = 0;
    }
    
    // 保存锚点信息
    if (getChildCount() > 0) {
        state.mAnchorPosition = mLastLayoutFromEnd ? getLastChildPosition() : getFirstChildPosition();
        state.mVisibleAnchorPosition = findFirstVisibleItemPositionInt();
        state.mSpanOffsetsSize = mSpanCount;
        state.mSpanOffsets = new int[mSpanCount];
        for (int i = 0; i < mSpanCount; i++) {
            state.mSpanOffsets[i] = mLastLayoutFromEnd ?
                    mSpans[i].getEndLine() - mPrimaryOrientation.getEndAfterPadding() :
                    mSpans[i].getStartLine() - mPrimaryOrientation.getStartAfterPadding();
        }
    } else {
        state.mAnchorPosition = RecyclerView.NO_POSITION;
        state.mVisibleAnchorPosition = RecyclerView.NO_POSITION;
        state.mSpanOffsetsSize = 0;
    }
    
    return state;
}

@Override
public void onRestoreInstanceState(Parcelable state) {
    if (state instanceof SavedState) {
        mPendingSavedState = (SavedState) state;
        requestLayout();
    }
}
```

## 4. 滚动处理

### 4.1 滚动方法

```java
@Override
public int scrollVerticallyBy(int dy, RecyclerView.Recycler recycler, RecyclerView.State state) {
    return scrollBy(dy, recycler, state);
}

@Override
public int scrollHorizontallyBy(int dx, RecyclerView.Recycler recycler, RecyclerView.State state) {
    return scrollBy(dx, recycler, state);
}

private int scrollBy(int dt, RecyclerView.Recycler recycler, RecyclerView.State state) {
    if (getChildCount() == 0 || dt == 0) {
        return 0;
    }
    
    // 准备布局状态
    mLayoutState.mRecycle = true;
    ensureOrientationHelper();
    
    final int layoutDirection = dt > 0 ? LayoutState.LAYOUT_END : LayoutState.LAYOUT_START;
    final int absDt = Math.abs(dt);
    
    // 更新布局状态
    updateLayoutState(layoutDirection, absDt, true, state);
    
    // 填充新的可见区域
    final int consumed = mLayoutState.mScrollingOffset +
            fill(recycler, mLayoutState, state);
    
    // 无法消耗滚动距离
    if (consumed < 0) {
        return 0;
    }
    
    // 实际滚动距离
    final int scrolled = absDt > consumed ? layoutDirection * consumed : dt;
    
    // 执行实际滚动
    mPrimaryOrientation.offsetChildren(-scrolled);
    
    return scrolled;
}
```

### 4.2 滚动到指定位置

```java
@Override
public void scrollToPosition(int position) {
    if (mPendingSavedState != null && mPendingSavedState.mAnchorPosition != position) {
        mPendingSavedState.invalidateAnchorPositionInfo();
    }
    
    mPendingScrollPosition = position;
    mPendingScrollPositionOffset = INVALID_OFFSET;
    requestLayout();
}

@Override
public void smoothScrollToPosition(RecyclerView recyclerView, RecyclerView.State state, int position) {
    LinearSmoothScroller scroller = new LinearSmoothScroller(recyclerView.getContext());
    scroller.setTargetPosition(position);
    startSmoothScroll(scroller);
}

@Override
public PointF computeScrollVectorForPosition(int targetPosition) {
    final int direction = calculateScrollDirectionForPosition(targetPosition);
    PointF outVector = new PointF();
    if (direction == 0) {
        return null;
    }
    
    if (mOrientation == HORIZONTAL) {
        outVector.x = direction;
        outVector.y = 0;
    } else {
        outVector.x = 0;
        outVector.y = direction;
    }
    
    return outVector;
}
```

## 5. 高级特性

### 5.1 懒加载跨度查找

StaggeredGridLayoutManager使用LazySpanLookup类来跟踪每个位置的跨度分配情况：

```java
static class LazySpanLookup {
    int[] mData;
    List<FullSpanItem> mFullSpanItems;
    
    /**
     * 获取指定位置的跨度索引
     */
    int getSpan(int position) {
        if (mData == null || position >= mData.length) {
            return INVALID_SPAN_ID;
        }
        return mData[position];
    }
    
    /**
     * 设置指定位置的跨度索引
     */
    void setSpan(int position, Span span) {
        ensureSize(position);
        mData[position] = span.mIndex;
    }
    
    // 其他方法...
}
```

### 5.2 锚点信息

StaggeredGridLayoutManager使用AnchorInfo来存储布局锚点信息：

```java
private class AnchorInfo {
    // 锚点位置
    int mPosition;
    // 锚点偏移量
    int mOffset;
    // 是否从末尾开始布局
    boolean mLayoutFromEnd;
    // 是否有效
    boolean mValid;
    
    // 重置锚点信息
    void reset() {
        mPosition = RecyclerView.NO_POSITION;
        mOffset = INVALID_OFFSET;
        mLayoutFromEnd = false;
        mValid = false;
    }
}
```

## 6. 性能优化

StaggeredGridLayoutManager相比其他LayoutManager实现更为复杂，布局计算也更耗时。因此，源码中包含了多种性能优化手段：

### 6.1 缓存机制

缓存每个跨度的起始和结束位置，减少重复计算：

```java
int getStartLine(int def) {
    if (mCachedStart != Integer.MIN_VALUE) {
        return mCachedStart;
    }
    
    if (mViews.size() == 0) {
        return def;
    }
    
    calculateCachedStart();
    return mCachedStart;
}

private void calculateCachedStart() {
    View startView = mViews.get(0);
    LayoutParams lp = getLayoutParams(startView);
    mCachedStart = mPrimaryOrientation.getDecoratedStart(startView);
}
```

### 6.2 预取机制

利用RecyclerView的预取机制，提前加载即将显示的Item：

```java
@Override
public void collectAdjacentPrefetchPositions(int dx, int dy, RecyclerView.State state,
        LayoutPrefetchRegistry layoutPrefetchRegistry) {
    int delta = (mOrientation == HORIZONTAL) ? dx : dy;
    if (getChildCount() == 0 || delta == 0) {
        return;
    }
    
    int layoutDir = delta > 0 ? LayoutState.LAYOUT_END : LayoutState.LAYOUT_START;
    final int absDy = Math.abs(delta);
    int containerSize = getMainDirSpec(absDy);
    
    // 预取即将显示的位置
    collectPrefetchPositionsForLayoutState(state, mLayoutState, layoutPrefetchRegistry);
}
```

## 总结

StaggeredGridLayoutManager是RecyclerView中最复杂的内置LayoutManager，主要用于实现瀑布流效果。它基于跨度(Span)概念，每个Item占据一个跨度，但高度或宽度可以不同，形成错落有致的布局效果。

相比LinearLayoutManager和GridLayoutManager，StaggeredGridLayoutManager实现更为复杂，布局计算也更耗时。因此，在使用时应注意性能问题，避免在Adapter中频繁变更Item的高度，这会导致布局重新计算。

StaggeredGridLayoutManager的主要特点包括：
1. 支持垂直和水平两种布局方向
2. 支持Item占据全部跨度(Full Span)
3. 提供多种间隙处理策略
4. 实现状态保存与恢复
5. 通过缓存机制优化性能

通过深入理解StaggeredGridLayoutManager的源码实现，我们可以更好地使用它来创建美观的瀑布流布局，同时避免常见的性能陷阱。 