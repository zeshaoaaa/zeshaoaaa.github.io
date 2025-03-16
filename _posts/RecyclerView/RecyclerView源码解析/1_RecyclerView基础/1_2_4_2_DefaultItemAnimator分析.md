# DefaultItemAnimator分析

DefaultItemAnimator是RecyclerView默认使用的ItemAnimator实现类。它提供了添加、删除、移动和更改Item时的默认动画效果。本文将深入分析DefaultItemAnimator的源码实现，了解RecyclerView默认动画的工作原理。

## 1. DefaultItemAnimator继承结构

DefaultItemAnimator继承自SimpleItemAnimator，而SimpleItemAnimator又继承自ItemAnimator。

```java
public class DefaultItemAnimator extends SimpleItemAnimator {
    // 实现代码
}
```

SimpleItemAnimator是对ItemAnimator的一层封装，提供了一些辅助方法，简化了动画实现的复杂度。DefaultItemAnimator通过继承SimpleItemAnimator，只需要实现具体的动画效果，而无需处理复杂的动画调度逻辑。

## 2. 动画实现原理

DefaultItemAnimator实现了以下几种典型的动画：

- **添加动画**：当一个ViewHolder被添加到RecyclerView时，从透明到完全显示并伴随轻微的放大效果
- **删除动画**：当一个ViewHolder从RecyclerView中移除时，从完全显示到透明并伴随轻微的缩小效果
- **移动动画**：当一个ViewHolder位置变化时，使用平移动画从旧位置移动到新位置
- **更改动画**：当一个ViewHolder内容变化时，旧的ViewHolder淡出，新的ViewHolder淡入

### 2.1 添加动画

```java
@Override
public boolean animateAdd(final RecyclerView.ViewHolder holder) {
    resetAnimation(holder);
    holder.itemView.setAlpha(0f);
    mPendingAdditions.add(holder);
    return true;
}

private void animateAddImpl(final RecyclerView.ViewHolder holder) {
    final View view = holder.itemView;
    final ViewPropertyAnimator animation = view.animate();
    mAddAnimations.add(holder);
    animation.alpha(1f).setDuration(mAddDuration)
            .setListener(new AnimatorListenerAdapter() {
                @Override
                public void onAnimationStart(Animator animator) {
                    dispatchAddStarting(holder);
                }
                @Override
                public void onAnimationCancel(Animator animator) {
                    view.setAlpha(1f);
                }
                @Override
                public void onAnimationEnd(Animator animator) {
                    animation.setListener(null);
                    dispatchAddFinished(holder);
                    mAddAnimations.remove(holder);
                    dispatchFinishedWhenDone();
                }
            }).start();
}
```

### 2.2 删除动画

```java
@Override
public boolean animateRemove(final RecyclerView.ViewHolder holder) {
    resetAnimation(holder);
    mPendingRemovals.add(holder);
    return true;
}

private void animateRemoveImpl(final RecyclerView.ViewHolder holder) {
    final View view = holder.itemView;
    final ViewPropertyAnimator animation = view.animate();
    mRemoveAnimations.add(holder);
    animation.setDuration(mRemoveDuration).alpha(0)
            .setListener(new AnimatorListenerAdapter() {
                @Override
                public void onAnimationStart(Animator animator) {
                    dispatchRemoveStarting(holder);
                }
                @Override
                public void onAnimationEnd(Animator animator) {
                    animation.setListener(null);
                    view.setAlpha(1);
                    dispatchRemoveFinished(holder);
                    mRemoveAnimations.remove(holder);
                    dispatchFinishedWhenDone();
                }
            }).start();
}
```

### 2.3 移动动画

```java
@Override
public boolean animateMove(final RecyclerView.ViewHolder holder, int fromX, int fromY,
        int toX, int toY) {
    final View view = holder.itemView;
    fromX += (int) holder.itemView.getTranslationX();
    fromY += (int) holder.itemView.getTranslationY();
    resetAnimation(holder);
    int deltaX = toX - fromX;
    int deltaY = toY - fromY;
    if (deltaX == 0 && deltaY == 0) {
        dispatchMoveFinished(holder);
        return false;
    }
    if (deltaX != 0) {
        view.setTranslationX(-deltaX);
    }
    if (deltaY != 0) {
        view.setTranslationY(-deltaY);
    }
    mPendingMoves.add(new MoveInfo(holder, fromX, fromY, toX, toY));
    return true;
}

private void animateMoveImpl(final RecyclerView.ViewHolder holder, int fromX, int fromY,
        int toX, int toY) {
    final View view = holder.itemView;
    final int deltaX = toX - fromX;
    final int deltaY = toY - fromY;
    if (deltaX != 0) {
        view.animate().translationX(0);
    }
    if (deltaY != 0) {
        view.animate().translationY(0);
    }
    
    final ViewPropertyAnimator animation = view.animate();
    mMoveAnimations.add(holder);
    animation.setDuration(mMoveDuration)
            .setListener(new AnimatorListenerAdapter() {
                @Override
                public void onAnimationStart(Animator animator) {
                    dispatchMoveStarting(holder);
                }
                @Override
                public void onAnimationCancel(Animator animator) {
                    if (deltaX != 0) {
                        view.setTranslationX(0);
                    }
                    if (deltaY != 0) {
                        view.setTranslationY(0);
                    }
                }
                @Override
                public void onAnimationEnd(Animator animator) {
                    animation.setListener(null);
                    dispatchMoveFinished(holder);
                    mMoveAnimations.remove(holder);
                    dispatchFinishedWhenDone();
                }
            }).start();
}
```

### 2.4 更改动画

```java
@Override
public boolean animateChange(RecyclerView.ViewHolder oldHolder,
        RecyclerView.ViewHolder newHolder, int fromX, int fromY, int toX, int toY) {
    if (oldHolder == newHolder) {
        // If oldHolder and newHolder are the same, run a simple animation to update the
        // item's position and size
        return animateMove(oldHolder, fromX, fromY, toX, toY);
    }
    
    final float prevTranslationX = oldHolder.itemView.getTranslationX();
    final float prevTranslationY = oldHolder.itemView.getTranslationY();
    final float prevAlpha = oldHolder.itemView.getAlpha();
    resetAnimation(oldHolder);
    int deltaX = (int) (toX - fromX - prevTranslationX);
    int deltaY = (int) (toY - fromY - prevTranslationY);
    
    // Recover previous translation state after ending animation
    oldHolder.itemView.setTranslationX(prevTranslationX);
    oldHolder.itemView.setTranslationY(prevTranslationY);
    oldHolder.itemView.setAlpha(prevAlpha);
    
    if (newHolder != null) {
        // carry over translation values
        resetAnimation(newHolder);
        newHolder.itemView.setTranslationX(-deltaX);
        newHolder.itemView.setTranslationY(-deltaY);
        newHolder.itemView.setAlpha(0);
    }
    
    mPendingChanges.add(new ChangeInfo(oldHolder, newHolder, fromX, fromY, toX, toY));
    return true;
}

private void animateChangeImpl(final ChangeInfo changeInfo) {
    final RecyclerView.ViewHolder holder = changeInfo.oldHolder;
    final View view = holder == null ? null : holder.itemView;
    final RecyclerView.ViewHolder newHolder = changeInfo.newHolder;
    final View newView = newHolder != null ? newHolder.itemView : null;
    
    if (view != null) {
        final ViewPropertyAnimator oldViewAnim = view.animate().setDuration(
                mChangeDuration);
        mChangeAnimations.add(changeInfo.oldHolder);
        oldViewAnim.translationX(changeInfo.toX - changeInfo.fromX);
        oldViewAnim.translationY(changeInfo.toY - changeInfo.fromY);
        oldViewAnim.alpha(0).setListener(new AnimatorListenerAdapter() {
            @Override
            public void onAnimationStart(Animator animator) {
                dispatchChangeStarting(changeInfo.oldHolder, true);
            }

            @Override
            public void onAnimationEnd(Animator animator) {
                oldViewAnim.setListener(null);
                view.setAlpha(1);
                view.setTranslationX(0);
                view.setTranslationY(0);
                dispatchChangeFinished(changeInfo.oldHolder, true);
                mChangeAnimations.remove(changeInfo.oldHolder);
                dispatchFinishedWhenDone();
            }
        }).start();
    }

    if (newView != null) {
        final ViewPropertyAnimator newViewAnimation = newView.animate();
        mChangeAnimations.add(changeInfo.newHolder);
        newViewAnimation.translationX(0).translationY(0).setDuration(mChangeDuration)
                .alpha(1).setListener(new AnimatorListenerAdapter() {
                    @Override
                    public void onAnimationStart(Animator animator) {
                        dispatchChangeStarting(changeInfo.newHolder, false);
                    }
                    @Override
                    public void onAnimationEnd(Animator animator) {
                        newViewAnimation.setListener(null);
                        newView.setAlpha(1);
                        newView.setTranslationX(0);
                        newView.setTranslationY(0);
                        dispatchChangeFinished(changeInfo.newHolder, false);
                        mChangeAnimations.remove(changeInfo.newHolder);
                        dispatchFinishedWhenDone();
                    }
                }).start();
    }
}
```

## 3. 动画调度机制

DefaultItemAnimator内部维护了多个列表来跟踪不同类型的动画：

```java
private ArrayList<RecyclerView.ViewHolder> mPendingRemovals = new ArrayList<>();
private ArrayList<RecyclerView.ViewHolder> mPendingAdditions = new ArrayList<>();
private ArrayList<MoveInfo> mPendingMoves = new ArrayList<>();
private ArrayList<ChangeInfo> mPendingChanges = new ArrayList<>();

private ArrayList<ArrayList<RecyclerView.ViewHolder>> mAdditionsList = new ArrayList<>();
private ArrayList<ArrayList<MoveInfo>> mMovesList = new ArrayList<>();
private ArrayList<ArrayList<ChangeInfo>> mChangesList = new ArrayList<>();

private ArrayList<RecyclerView.ViewHolder> mAddAnimations = new ArrayList<>();
private ArrayList<RecyclerView.ViewHolder> mMoveAnimations = new ArrayList<>();
private ArrayList<RecyclerView.ViewHolder> mRemoveAnimations = new ArrayList<>();
private ArrayList<RecyclerView.ViewHolder> mChangeAnimations = new ArrayList<>();
```

这些列表的作用如下：

- **mPendingXXX**：待处理的动画操作
- **mXXXList**：分批执行的动画操作列表
- **mXXXAnimations**：正在执行的动画

DefaultItemAnimator的动画调度主要通过`runPendingAnimations()`方法实现：

```java
@Override
public void runPendingAnimations() {
    boolean removalsPending = !mPendingRemovals.isEmpty();
    boolean movesPending = !mPendingMoves.isEmpty();
    boolean changesPending = !mPendingChanges.isEmpty();
    boolean additionsPending = !mPendingAdditions.isEmpty();
    
    if (!removalsPending && !movesPending && !additionsPending && !changesPending) {
        // nothing to animate
        return;
    }
    
    // 先执行所有移除动画
    for (RecyclerView.ViewHolder holder : mPendingRemovals) {
        animateRemoveImpl(holder);
    }
    mPendingRemovals.clear();
    
    // 接着执行移动动画
    if (movesPending) {
        final ArrayList<MoveInfo> moves = new ArrayList<>();
        moves.addAll(mPendingMoves);
        mMovesList.add(moves);
        mPendingMoves.clear();
        Runnable mover = new Runnable() {
            @Override
            public void run() {
                for (MoveInfo moveInfo : moves) {
                    animateMoveImpl(moveInfo.holder, moveInfo.fromX, moveInfo.fromY,
                            moveInfo.toX, moveInfo.toY);
                }
                moves.clear();
                mMovesList.remove(moves);
            }
        };
        if (removalsPending) {
            View view = moves.get(0).holder.itemView;
            ViewCompat.postOnAnimationDelayed(view, mover, getRemoveDuration());
        } else {
            mover.run();
        }
    }
    
    // 然后执行变化动画
    if (changesPending) {
        // ...类似移动动画的处理
    }
    
    // 最后执行添加动画
    if (additionsPending) {
        // ...类似移动动画的处理
    }
}
```

动画执行顺序为：

1. 先执行**删除**动画
2. 然后是**移动**动画
3. 接着是**更改**动画
4. 最后是**添加**动画

这种顺序设计可以保证动画效果的自然和流畅。

## 4. 自定义DefaultItemAnimator

我们可以通过继承DefaultItemAnimator来自定义RecyclerView的动画效果：

```java
public class CustomItemAnimator extends DefaultItemAnimator {
    @Override
    public boolean animateAdd(RecyclerView.ViewHolder holder) {
        // 重新设置初始状态
        holder.itemView.setAlpha(0f);
        holder.itemView.setScaleX(0.5f);
        holder.itemView.setScaleY(0.5f);
        
        return super.animateAdd(holder);
    }

    @Override
    protected void animateAddImpl(final RecyclerView.ViewHolder holder) {
        final View view = holder.itemView;
        final ViewPropertyAnimator animation = view.animate();
        
        animation.alpha(1f)
                .scaleX(1f)
                .scaleY(1f)
                .setDuration(300)
                .setInterpolator(new OvershootInterpolator(1.5f))
                .setListener(new AnimatorListenerAdapter() {
                    @Override
                    public void onAnimationStart(Animator animator) {
                        dispatchAddStarting(holder);
                    }

                    @Override
                    public void onAnimationCancel(Animator animator) {
                        view.setAlpha(1f);
                        view.setScaleX(1f);
                        view.setScaleY(1f);
                    }

                    @Override
                    public void onAnimationEnd(Animator animator) {
                        animation.setListener(null);
                        dispatchAddFinished(holder);
                        mAddAnimations.remove(holder);
                        dispatchFinishedWhenDone();
                    }
                }).start();
    }
    
    // 可以重写其他方法以自定义不同的动画效果
}
```

## 5. 性能考量

DefaultItemAnimator使用Android的属性动画系统(Property Animation)实现动画效果。虽然属性动画性能良好，但在处理大量Item同时动画时，仍然需要注意以下几点：

1. **降低动画时长**：当需要同时处理大量动画时，可以适当降低动画时长
2. **简化动画效果**：减少同时使用的动画属性
3. **分批处理**：DefaultItemAnimator内部已实现分批处理机制
4. **避免频繁刷新**：尽量批量更新数据，避免频繁触发动画

## 总结

DefaultItemAnimator作为RecyclerView的默认动画实现，提供了简单而优雅的Item动画效果。它采用四步动画执行顺序（删除、移动、更改、添加），保证动画的自然流畅。通过继承DefaultItemAnimator，开发者可以轻松自定义RecyclerView的动画效果，提升应用的用户体验。 