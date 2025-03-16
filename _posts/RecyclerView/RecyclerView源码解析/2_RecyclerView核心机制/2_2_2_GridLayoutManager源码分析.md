# GridLayoutManager源码分析

## 1. 概述

GridLayoutManager是RecyclerView中用于实现网格布局的LayoutManager，它继承自LinearLayoutManager，在保留线性布局基本功能的同时，增加了对网格布局的支持。本文将深入分析GridLayoutManager的源码实现，探究其工作原理、核心算法以及跨度（span）管理机制。



## 2. 类继承结构

GridLayoutManager继承自LinearLayoutManager，这使得它可以复用线性布局的大部分逻辑，同时通过重写关键方法来实现网格布局：

```java
public class GridLayoutManager extends LinearLayoutManager {
    // 网格特有的属性和方法
}
```

通过继承关系，GridLayoutManager获得了LinearLayoutManager的所有功能，包括方向控制、滚动处理、视图回收等，只需专注于实现网格特有的布局逻辑。



## 3. 核心属性

GridLayoutManager具有以下核心属性：

```java
public class GridLayoutManager extends LinearLayoutManager {
    // 默认跨度数
    boolean mUsingSpansToEstimateScrollBarDimensions;
    // 总跨度数
    int mSpanCount = -1;
    // 用于存储每个位置所占跨度的数组
    int[] mSpanSizeLookup;
    // 可见视图跨度分布
    View[] mSet;
    // 跨度管理器
    SpanSizeLookup mSpanSizeLookup = new DefaultSpanSizeLookup();
    
    // 默认跨度管理器实现
    public static final class DefaultSpanSizeLookup extends SpanSizeLookup {
        @Override
        public int getSpanSize(int position) {
            return 1; // 默认每个item占用1个跨度
        }
    }
}
```



## 4. 构造函数

GridLayoutManager提供了两个主要构造函数：

```java
// 简单构造，指定跨度数
public GridLayoutManager(Context context, int spanCount) {
    super(context);
    setSpanCount(spanCount);
}

// 完整构造，可指定方向和是否反向布局
public GridLayoutManager(Context context, int spanCount, 
                        @RecyclerView.Orientation int orientation,
                        boolean reverseLayout) {
    super(context, orientation, reverseLayout);
    setSpanCount(spanCount);
}

// 设置跨度数的方法
public void setSpanCount(int spanCount) {
    if (spanCount == mSpanCount) {
        return;
    }
    mSpanCount = spanCount;
    mSpanSizeLookup.invalidateSpanIndexCache();
    requestLayout();
}
```



## 5. SpanSizeLookup机制

SpanSizeLookup是GridLayoutManager中最核心的组件之一，它决定了每个位置的item应该占据多少跨度：

```java
public abstract static class SpanSizeLookup {
    // 缓存机制
    private boolean mCacheSpanIndices = false;
    final SparseIntArray mSpanIndexCache = new SparseIntArray();
    
    // 核心方法：获取指定位置的跨度大小
    public abstract int getSpanSize(int position);
    
    // 获取指定位置的跨度索引
    public int getSpanIndex(int position, int spanCount) {
        // 计算当前位置item的起始跨度索引
        int positionSpanSize = getSpanSize(position);
        if (positionSpanSize == spanCount) {
            return 0; // 如果占满整行，则从0开始
        }
        
        // 否则需要计算前面已占用的跨度
        int span = 0;
        int startPos = 0;
        // 尝试从缓存获取
        if (mCacheSpanIndices && mSpanIndexCache.size() > 0) {
            // 计算前一个有效位置
            int prevKey = findReferenceIndexFromCache(position);
            if (prevKey >= 0) {
                span = mSpanIndexCache.get(prevKey) + getSpanSize(prevKey);
                startPos = prevKey + 1;
            }
        }
        
        // 从startPos开始计算到position的跨度
        for (int i = startPos; i < position; i++) {
            int size = getSpanSize(i);
            span += size;
            if (span == spanCount) {
                span = 0;
            } else if (span > spanCount) {
                // 如果超出了一行，重新计算
                span = size;
            }
        }
        
        // 确保不超出总跨度
        if (span + positionSpanSize <= spanCount) {
            return span;
        }
        return 0; // 如果放不下，换到下一行
    }
    
    // 获取指定位置所在的组（行或列）
    public int getSpanGroupIndex(int adapterPosition, int spanCount) {
        int span = 0;
        int group = 0;
        
        for (int i = 0; i < adapterPosition; i++) {
            int size = getSpanSize(i);
            span += size;
            if (span == spanCount) {
                span = 0;
                group++;
            } else if (span > spanCount) {
                // 如果超出了一行，重新计算
                span = size;
                group++;
            }
        }
        
        // 检查当前位置是否需要新的一组
        span += getSpanSize(adapterPosition);
        if (span > spanCount) {
            group++;
        }
        
        return group;
    }
}
```



## 6. 布局流程分析

### 6.1 onLayoutChildren方法

GridLayoutManager重写了LinearLayoutManager的onLayoutChildren方法，实现网格布局的核心逻辑：

```java
@Override
public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
    if (mPendingSavedState != null || mPendingScrollPosition != RecyclerView.NO_POSITION) {
        if (state.getItemCount() == 0) {
            removeAndRecycleAllViews(recycler);
            return;
        }
    }
    
    // 保存当前布局信息，以便在需要时恢复
    if (mPendingSavedState != null) {
        // 恢复保存的状态
    }
    
    // 准备布局参数
    ensureLayoutState();
    mLayoutState.mRecycle = false;
    
    // 决定布局方向
    resolveShouldLayoutReverse();
    
    View focused = getFocusedChild();
    if (!mAnchorInfo.mValid || mPendingScrollPosition != RecyclerView.NO_POSITION
            || mPendingSavedState != null) {
        mAnchorInfo.reset();
        mAnchorInfo.mLayoutFromEnd = mShouldReverseLayout;
        // 计算布局锚点
        updateAnchorInfoForLayout(recycler, state, mAnchorInfo);
        mAnchorInfo.mValid = true;
    }
    
    // 执行实际布局
    detachAndScrapAttachedViews(recycler);
    mLayoutState.mInfinite = resolveIsInfinite();
    mLayoutState.mIsPreLayout = state.isPreLayout();
    mLayoutState.mNoRecycleSpace = 0;
    
    if (mAnchorInfo.mLayoutFromEnd) {
        // 从末尾开始布局
        updateLayoutStateToFillStart(mAnchorInfo);
        mLayoutState.mExtraFillSpace = mExtraLayoutSpace;
        fill(recycler, mLayoutState, state, false);
        // ...
    } else {
        // 从开始处布局
        updateLayoutStateToFillEnd(mAnchorInfo);
        mLayoutState.mExtraFillSpace = mExtraLayoutSpace;
        fill(recycler, mLayoutState, state, false);
        // ...
    }
    
    // 处理剩余空间
    // ...
    
    if (!state.isPreLayout()) {
        mOrientationHelper.onLayoutComplete();
    } else {
        mAnchorInfo.reset();
    }
    mLastStackFromEnd = mStackFromEnd;
}
```



### 6.2 fill方法

GridLayoutManager重写了LinearLayoutManager的fill方法，实现了网格布局的填充逻辑：

```java
@Override
int fill(RecyclerView.Recycler recycler, LayoutState layoutState,
        RecyclerView.State state, boolean stopOnFocusable) {
    // 记录开始位置和剩余空间
    int remainingSpan = mSpanCount;
    int start = layoutState.mAvailable;
    
    if (mLayoutState.mScrollingOffset != LayoutState.SCROLLING_OFFSET_NaN) {
        if (layoutState.mAvailable < 0) {
            layoutState.mScrollingOffset += layoutState.mAvailable;
        }
        recycleByLayoutState(recycler, layoutState);
    }
    
    // 剩余可用空间
    int remainingSpace = layoutState.mAvailable + layoutState.mExtraFillSpace;
    
    // 用于记录每个跨度位置的视图
    if (mSet == null || mSet.length < mSpanCount) {
        mSet = new View[mSpanCount];
    }
    
    // 开始填充
    while ((layoutState.hasMore(state) && remainingSpace > 0)) {
        // 准备当前行/列的布局
        int count = 0;
        int consumedSpanCount = 0;
        
        // 填充一行（或一列）
        for (int i = 0; i < mSpanCount && layoutState.hasMore(state); i++) {
            int pos = layoutState.mCurrentPosition;
            int spanSize = mSpanSizeLookup.getSpanSize(pos);
            if (spanSize > mSpanCount) {
                spanSize = mSpanCount;
            }
            
            // 如果当前行剩余空间不足，移到下一行
            if (remainingSpan - spanSize < 0) {
                break;
            }
            
            remainingSpan -= spanSize;
            
            // 获取并添加视图
            View view = layoutState.next(recycler);
            if (view == null) {
                break;
            }
            
            mSet[count] = view;
            count++;
            consumedSpanCount += spanSize;
        }
        
        if (count == 0) {
            break; // 如果没有添加任何视图，退出循环
        }
        
        // 计算并分配空间给当前行/列的视图
        assignSpans(layoutState, count, consumedSpanCount);
        
        // 执行实际布局
        for (int i = 0; i < count; i++) {
            View view = mSet[i];
            mLayoutState.mScrapList.add(view);
            addView(view);
            measureChildWithMargins(view, 0, 0);
            layoutDecoratedWithMargins(view, ...);
            // ...
        }
        
        // 更新剩余空间和位置
        remainingSpace -= layoutChunkResult.mConsumed;
        layoutState.mAvailable -= layoutChunkResult.mConsumed;
        remainingSpan = mSpanCount; // 重置跨度计数
    }
    
    // 清理
    Arrays.fill(mSet, null);
    return start - layoutState.mAvailable;
}
```



### 6.3 assignSpans方法

assignSpans方法负责将位置分配给每个视图，这是GridLayoutManager实现网格布局的关键：

```java
void assignSpans(RecyclerView.Recycler recycler, RecyclerView.State state,
        int count, int consumedSpanCount, boolean layingOutInPrimaryDirection) {
    // 计算每个位置的跨度索引
    if (layingOutInPrimaryDirection) {
        // 从左到右或从上到下
        for (int i = 0; i < count; i++) {
            View view = mSet[i];
            int position = getPosition(view);
            int spanIndex = mSpanSizeLookup.getSpanIndex(position, mSpanCount);
            int spanSize = mSpanSizeLookup.getSpanSize(position);
            
            // 设置布局参数
            LayoutParams lp = (LayoutParams) view.getLayoutParams();
            lp.mSpanIndex = spanIndex;
            lp.mSpanSize = spanSize;
        }
    } else {
        // 从右到左或从下到上
        // 类似逻辑，但方向相反
    }
}
```



## 7. 跨度分组计算

GridLayoutManager通过SpanSizeLookup的getSpanGroupIndex方法计算每个位置所在的组（行或列）：

```java
// 计算一个位置所在的组索引
public int getSpanGroupIndex(int position, int spanCount) {
    // 初始化
    int span = 0;
    int group = 0;
    int positionSpanSize = getSpanSize(position);
    
    // 遍历前面的所有位置
    for (int i = 0; i < position; i++) {
        int size = getSpanSize(i);
        span += size;
        
        // 如果刚好填满一行
        if (span == spanCount) {
            span = 0;
            group++;
        } 
        // 如果超出一行，需要换行
        else if (span > spanCount) {
            span = size;
            group++;
        }
    }
    
    // 检查当前位置是否需要新的组
    if (span + positionSpanSize > spanCount) {
        group++;
    }
    
    return group;
}
```



## 8. 自定义SpanSizeLookup

GridLayoutManager允许通过自定义SpanSizeLookup实现复杂的网格布局：

```java
// 示例：标题占据整行，内容每行两个
public class HeaderSpanSizeLookup extends GridLayoutManager.SpanSizeLookup {
    private final int mSpanCount;
    
    public HeaderSpanSizeLookup(int spanCount) {
        mSpanCount = spanCount;
    }
    
    @Override
    public int getSpanSize(int position) {
        if (isHeader(position)) {
            return mSpanCount; // 标题占满整行
        } else {
            return 1; // 内容项占1格
        }
    }
    
    private boolean isHeader(int position) {
        return position % 5 == 0; // 每5个item一个标题
    }
}

// 使用方式
GridLayoutManager layoutManager = new GridLayoutManager(context, 2);
layoutManager.setSpanSizeLookup(new HeaderSpanSizeLookup(2));
recyclerView.setLayoutManager(layoutManager);
```



## 9. 滚动处理

GridLayoutManager复用了LinearLayoutManager的滚动机制，并在必要处添加了针对网格布局的调整：

```java
@Override
public int scrollVerticallyBy(int dy, RecyclerView.Recycler recycler, RecyclerView.State state) {
    return super.scrollVerticallyBy(dy, recycler, state);
}

@Override
public int scrollHorizontallyBy(int dx, RecyclerView.Recycler recycler, RecyclerView.State state) {
    return super.scrollHorizontallyBy(dx, recycler, state);
}
```



## 10. 布局参数

GridLayoutManager定义了自己的LayoutParams类，扩展了LinearLayoutManager的LayoutParams：

```java
public static class LayoutParams extends RecyclerView.LayoutParams {
    // 在网格中的跨度索引
    int mSpanIndex;
    // 占用的跨度数
    int mSpanSize;
    
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
    
    // 获取跨度索引
    public int getSpanIndex() {
        return mSpanIndex;
    }
    
    // 获取跨度大小
    public int getSpanSize() {
        return mSpanSize;
    }
}
```



## 11. 状态保存与恢复

GridLayoutManager实现了状态的保存与恢复，确保在配置变更或应用重启时保持滚动位置：

```java
@Override
public Parcelable onSaveInstanceState() {
    if (mPendingSavedState != null) {
        return new SavedState(mPendingSavedState);
    }
    SavedState state = new SavedState();
    state.mAnchorPosition = mPendingScrollPosition != RecyclerView.NO_POSITION
            ? mPendingScrollPosition : getPosition(findFirstVisibleItemClosestToStart(!mShouldReverseLayout));
    state.mAnchorOffset = findFirstVisibleItemPositionOffset();
    state.mSpanCount = mSpanCount;
    return state;
}

@Override
public void onRestoreInstanceState(Parcelable state) {
    if (state instanceof SavedState) {
        mPendingSavedState = (SavedState) state;
        if (mPendingScrollPosition != RecyclerView.NO_POSITION) {
            mPendingSavedState.invalidateAnchorPositionInfo();
        }
        requestLayout();
    }
}

// 保存状态的内部类
public static class SavedState implements Parcelable {
    int mAnchorPosition;
    int mAnchorOffset;
    int mSpanCount;
    
    // ... Parcelable实现
}
```



## 12. 总结

GridLayoutManager是RecyclerView中实现网格布局的核心组件，通过以下机制实现其功能：

1. **继承LinearLayoutManager**：复用线性布局的基础逻辑，专注于网格特有的部分
2. **SpanSizeLookup**：灵活控制每个位置item的跨度大小
3. **跨度计算**：通过精确计算每个位置的跨度索引和组索引实现网格布局
4. **布局流程**：重写关键布局方法，支持不同方向和不同大小的item布局
5. **自定义LayoutParams**：通过扩展LayoutParams存储网格特有的布局信息

GridLayoutManager的设计体现了Android团队对复用和扩展性的重视，它在保持高性能的同时，提供了足够的灵活性来满足各种网格布局需求。通过深入理解其源码实现，开发者可以更好地利用和扩展这一强大组件。 