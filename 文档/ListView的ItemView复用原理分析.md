#   ListView的ItemView的复用原理分析

1.  ListView继承自AbsListView.GridView也继承自AbsListView.对于ItemView的复用逻辑,其实就是在AbsListView中处理的.
```
在AbsListView的obtainView中:

    View obtainView(int position, boolean[] outMetadata) {

        outMetadata[0] = false;

        //mRecycler是一个RecycleBin实例.可以将RecycleBin理解成一个View复用池.
        //这里其实就是通过position看是否可以获取到一个对应的临时状态的View
        final View transientView = mRecycler.getTransientStateView(position);

        //当临时状态的View不为空时.
        if (transientView != null) {
            final LayoutParams params = (LayoutParams) transientView.getLayoutParams();

            //如果viewType和当前postion需要的viewType一致.
            if (params.viewType == mAdapter.getItemViewType(position)) {
                //将transientView作为converView传递给adapter的getView方法
                final View updatedView = mAdapter.getView(position, transientView, this);

                //如果adapter返回的View不是上面找到的transientView.
                if (updatedView != transientView) {
                    //将位置postion相关的参数设置到updateView的layoutParams上.比如viewType,itemId等.
                    setItemViewLayoutParams(updatedView, position);
                    //将updateView添加到对应的view池中.
                    mRecycler.addScrapView(updatedView, position);
                }
            }

            outMetadata[0] = true;

            transientView.dispatchFinishTemporaryDetach();
            return transientView;
        }

        //看这里是否可以从复用池中获取对应的View
        final View scrapView = mRecycler.getScrapView(position);
        final View child = mAdapter.getView(position, scrapView, this);

        if (scrapView != null) {
            if (child != scrapView) {
                //adapter返回的view不是前面查找到的可复用view.添加到对应的view池.
                mRecycler.addScrapView(scrapView, position);
            } else if (child.isTemporarilyDetached()) {
                outMetadata[0] = true;
                child.dispatchFinishTemporaryDetach();
            }
        }
        .......
        //设置layoutParams.
        setItemViewLayoutParams(child, position);
        return child;
    }
```
2.  RecyclerBin的几个成员变量
```
    private ArrayList<View>[] mScrapViews;//当viewType个数大于1时,采用该View池.

    private int mViewTypeCount;//dapter返回的ViewType的个数

    private ArrayList<View> mCurrentScrap;//当viewType个数是1的时候,采用该View池.

    private SparseArray<View> mTransientStateViews;//临时状态的View池.例如动画中.
    private LongSparseArray<View> mTransientStateViewsById;//临时状态的View池.View和itemId绑定.
```

3.  RecyclerBin的addScrapView方法:
```
        //该方法的作用是将view放进View池中.等待复用.
        void addScrapView(View scrap, int position) {
            //如果View的LayoutParams为null.则不会将该View放进View池.
            final AbsListView.LayoutParams lp = (AbsListView.LayoutParams) scrap.getLayoutParams();
            if (lp == null) {
                return;
            }

            lp.scrappedFromPosition = position;

            // Remove but don't scrap header or footer views, or views that
            // should otherwise not be recycled.
            final int viewType = lp.viewType;
            //判断当前正在被缓存的View的ViewType是否>=0;否则不缓存.
            if (!shouldRecycleViewType(viewType)) {
                // Can't recycle. If it's not a header or footer, which have
                // special handling and should be ignored, then skip the scrap
                // heap and we'll fully detach the view later.
                if (viewType != ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {
                    getSkippedScrap().add(scrap);
                }
                return;
            }

            scrap.dispatchStartTemporaryDetach();

            //如果当前View是临时状态.则将View放入对应的临时状态View池.
            final boolean scrapHasTransientState = scrap.hasTransientState();
            if (scrapHasTransientState) {
                if (mAdapter != null && mAdapterHasStableIds) {
                    // If the adapter has stable IDs, we can reuse the view for
                    // the same data.
                    if (mTransientStateViewsById == null) {
                        mTransientStateViewsById = new LongSparseArray<>();
                    }
                    mTransientStateViewsById.put(lp.itemId, scrap);
                } else if (!mDataChanged) {
                    // If the data hasn't changed, we can reuse the views at
                    // their old positions.
                    if (mTransientStateViews == null) {
                        mTransientStateViews = new SparseArray<>();
                    }
                    mTransientStateViews.put(position, scrap);
                } else {
                    // Otherwise, we'll have to remove the view and start over.
                    getSkippedScrap().add(scrap);
                }
            } else {
                //不是临时状态.则根据viewType的总数.选择将View放入mCurrentScrap缓存池或者mScrapViews中.
                if (mViewTypeCount == 1) {
                    mCurrentScrap.add(scrap);
                } else {
                    mScrapViews[viewType].add(scrap);
                }

                if (mRecyclerListener != null) {
                    mRecyclerListener.onMovedToScrapHeap(scrap);
                }
            }
        }
```

4.  RecyclerBin的getTransientStateView方法:
```
        //根据positon.获取对应的临时状态的View.
        View getTransientStateView(int position) {
            if (mAdapter != null && mAdapterHasStableIds && mTransientStateViewsById != null) {
                long id = mAdapter.getItemId(position);
                //根据itemId获取对应的View
                View result = mTransientStateViewsById.get(id);
                mTransientStateViewsById.remove(id);
                return result;
            }
            if (mTransientStateViews != null) {
                final int index = mTransientStateViews.indexOfKey(position);
                if (index >= 0) {
                    //根据positon获取对应的View
                    View result = mTransientStateViews.valueAt(index);
                    mTransientStateViews.removeAt(index);
                    return result;
                }
            }
            return null;
        }
```

5.  RecyclerBin的getScrapView方法:
```
        View getScrapView(int position) {
            //查找positon所对于的viewType.
            final int whichScrap = mAdapter.getItemViewType(position);
            if (whichScrap < 0) {
                return null;
            }

            //如果viewType种类为1.则从mCurrentScrap池中查找.否则从mScrapViews对应的viewType对应的池中查找.
            if (mViewTypeCount == 1) {
                return retrieveFromScrap(mCurrentScrap, position);
            } else if (whichScrap < mScrapViews.length) {
                return retrieveFromScrap(mScrapViews[whichScrap], position);
            }
            return null;
        }
```

6.  RecyclerBin的retrieveFromScrap方法:
```
        private View retrieveFromScrap(ArrayList<View> scrapViews, int position) {
            final int size = scrapViews.size();
            if (size > 0) {
                // See if we still have a view for this position or ID.
                for (int i = 0; i < size; i++) {
                    final View view = scrapViews.get(i);
                    final AbsListView.LayoutParams params =
                            (AbsListView.LayoutParams) view.getLayoutParams();

                    //优先根据itemId或者view从哪个positon被放入缓存池的position查找view.
                    if (mAdapterHasStableIds) {
                        final long id = mAdapter.getItemId(position);
                        if (id == params.itemId) {
                            return scrapViews.remove(i);
                        }
                    } else if (params.scrappedFromPosition == position) {
                        final View scrap = scrapViews.remove(i);
                        clearAccessibilityFromScrap(scrap);
                        return scrap;
                    }
                }
                //如果上述方式没找到可复用的view.则直接取复用池中的最后一个.
                final View scrap = scrapViews.remove(size - 1);
                clearAccessibilityFromScrap(scrap);
                return scrap;
            } else {
                return null;
            }
        }
```