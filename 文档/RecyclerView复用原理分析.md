#   RecyclerView复用原理分析.
RecyclerView的复用机制完全由它的一个内部类Recycler类实现.

1. 先来看一下Recycler的一些成员变量.
```
        //ViewHolder一级缓存池.默认最大size为DEFAULT_CACHE_SIZE.也就是2.
        final ArrayList<ViewHolder> mCachedViews = new ArrayList<ViewHolder>();
        //ViewHolder二级缓存池.默认最大size为5.
        RecycledViewPool mRecyclerPool;//mRecyclerPool也可以通过RecyclerView.setRecycledViewPool设置.也可以通过recyclerView.getRecycledViewPool获取.
        int mViewCacheMax = DEFAULT_CACHE_SIZE;//mViewCacheMax可以通过RecyclerView.setItemViewCacheSize方法设置.
        static final int DEFAULT_CACHE_SIZE = 2;
        .....
        //mCachedViews和mRecyclerPool的区别是mCachedViews是作为快速缓存池.在mCachedViews找到的可复用的ViewHolder可以不用重新执行bind操作.
        //mRecyclerPool中查找的缓存需要重新bind.另外.可以从外部设置RecycledViewPool,跨RecyclerView共享一个缓存池.
        //当一个回收触发时,mCachedViews缓存满了.会将第一个缓存添加到mRecyclerPool中.然后再添加进mCachedViews尾部.
        //当mRecyclerPool满了.也不会在继续缓存.

```

2. 复用查找过程:Recycler的tryGetViewHolderForPositionByDeadline方法:
```
  @Nullable
        ViewHolder tryGetViewHolderForPositionByDeadline(int position,
                boolean dryRun, long deadlineNs) {
            ......
            boolean fromScrapOrHiddenOrCache = false;
            ViewHolder holder = null;
            .....

            //通过position查找
            if (holder == null) {
                holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
                if (holder != null) {
                    if (!validateViewHolderForOffsetPosition(holder)) {
                        //不可用
                        .....
                        holder = null;
                    } else {
                        fromScrapOrHiddenOrCache = true;
                    }
                }
            }

            if (holder == null) {
                final int offsetPosition = mAdapterHelper.findPositionOffset(position);
                ....
                //通过itemId查找
                final int type = mAdapter.getItemViewType(offsetPosition);
                if (mAdapter.hasStableIds()) {
                    //和上面通过postion查找逻辑差不多.
                    holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition),
                            type, dryRun);
                    if (holder != null) {
                        // update position
                        holder.mPosition = offsetPosition;
                        fromScrapOrHiddenOrCache = true;
                    }
                }
                ......

                //从RecycledViewPool中取查找对应type的ViewHolder
                if (holder == null) { // fallback to pool
                    holder = getRecycledViewPool().getRecycledView(type);

                    if (holder != null) {
                        //清除旧数据,因此通过RecyclerViewPool查找出来的缓存ViewHolder在后面需要重新执行bind操作.
                        holder.resetInternal();
                    }
                }


                //没找到可复用的ViewHolder.调用adapter的createViewHolder创建新的ViewHolder
                if (holder == null) {
                    long start = getNanoTime();
                    .....
                    holder = mAdapter.createViewHolder(RecyclerView.this, type);
                    .....
                    long end = getNanoTime();
                    mRecyclerPool.factorInCreateTime(type, end - start);

                }
            }

            ......

            boolean bound = false;
            if (mState.isPreLayout() && holder.isBound()) {
                holder.mPreLayoutPosition = position;
            } else if (!holder.isBound() || holder.needsUpdate() || holder.isInvalid()) {
                .....
                final int offsetPosition = mAdapterHelper.findPositionOffset(position);

                //没有bind过,需要执行bind方法.
                bound = tryBindViewHolderByDeadline(holder, offsetPosition, position, deadlineNs);
            }
            ....
            return holder;
        }
```

3.  Recycler的getScrapOrHiddenOrCachedHolderForPosition方法:
```
        ViewHolder getScrapOrHiddenOrCachedHolderForPosition(int position, boolean dryRun) {
            final int scrapCount = mAttachedScrap.size();
            .......

            // 从cache中查找,mCachedViews就是一个简单的列表容器
            final int cacheSize = mCachedViews.size();
            for (int i = 0; i < cacheSize; i++) {
                final ViewHolder holder = mCachedViews.get(i);
                if (!holder.isInvalid() && holder.getLayoutPosition() == position) {
                    if (!dryRun) {
                        mCachedViews.remove(i);
                    }
                    return holder;
                }
            }
            return null;
        }
```

4.  RecyclerViewPool的一些成员变量
```
    //默认缓存最大数量
    private static final int DEFAULT_MAX_SCRAP = 5;

    //缓存池.
    static class ScrapData {
        ArrayList<ViewHolder> mScrapHeap = new ArrayList<>();
        int mMaxScrap = DEFAULT_MAX_SCRAP;
        long mCreateRunningAverageNs = 0;
        long mBindRunningAverageNs = 0;
    }

    //已viewType为key的对应缓存池为value的缓存管理容器.
    SparseArray<ScrapData> mScrap = new SparseArray<>();
```

5.  RecyclerViewPool的getRecycledView方法:

```
        public ViewHolder getRecycledView(int viewType) {
            //从缓存容器中获取viewType对应的缓存池.
            final ScrapData scrapData = mScrap.get(viewType);
            //如果缓存池不为空,则从缓存池中取最后一个ViewHolder.
            if (scrapData != null && !scrapData.mScrapHeap.isEmpty()) {
                final ArrayList<ViewHolder> scrapHeap = scrapData.mScrapHeap;
                return scrapHeap.remove(scrapHeap.size() - 1);
            }
            return null;
        }
```