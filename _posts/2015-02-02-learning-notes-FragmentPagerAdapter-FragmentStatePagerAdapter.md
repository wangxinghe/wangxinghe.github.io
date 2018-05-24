---
layout: post
comments: true
title: "Android学习笔记——FragmentPagerAdapter/FragmentStatePagerAdapter"
description: "Fragment相关"
category: android
tags: [Android]
---

**1、ViewPager**    
**（1）页面填充populate**    
**（1）状态保存和恢复onSaveInstanceState/onRestoreInstanceState**    
**2、FragmentPagerAdapter／FragmentStatePagerAdapter差别**    
**3、参考文档**    


### 1、ViewPager    

ViewPager搭配PagerAdapter一起使用，实现页面的左右切换。    
PagerAdapter里的item可以是View（填充View list），也可以是fragment（使用FragmentPagerAdapter／FragmentStatePagerAdapter）

    /**属性**/
    private PagerAdapter mAdapter;
    //以mCurItem为中心的左右位置偏移
    private int mOffscreenPageLimit = DEFAULT_OFFSCREEN_PAGES;
    //当前显示的item位置
    private int mCurItem;
    
    /**方法**/
    setOffscreenPageLimit(int limit)
    public void setAdapter(PagerAdapter adapter)
    public void setCurrentItem(int item)
    //填充页面
    populate();
    //状态保存和恢复
    public Parcelable onSaveInstanceState()
    public void onRestoreInstanceState(Parcelable state)
    
#### （1）页面填充populate    

ViewPager状态发生变化时会调到populate()去填充页面，如setCurrentItem(item)或setAdapter()或dataSetChanged()时。

填充规则是：    
对于`mCurItem-mOffscreenPageLimit和mCurItem＋mOffscreenPageLimit`范围内的页面，会调用mAdapter.instantiateItem()初始化（如果没初始化的话）    
对于超出这个范围的页面，会调用mAdapter.destroyItem()进行销毁操作。    

#### （2）状态保存和恢复onSaveInstanceState/onRestoreInstanceState    

会在View的生命周期调用mAdapter.saveState()和mAdapter.restoreState()实现状态的保存和恢复。    

当然前提还是要看PagerAdapter在回调中有没有保存和恢复逻辑。

    //ViewPager
    
    @Override
    public Parcelable onSaveInstanceState() {
        Parcelable superState = super.onSaveInstanceState();
        SavedState ss = new SavedState(superState);
        ss.position = mCurItem;
        if (mAdapter != null) {
            ss.adapterState = mAdapter.saveState();
        }
        return ss;
    }

    @Override
    public void onRestoreInstanceState(Parcelable state) {
        if (!(state instanceof SavedState)) {
            super.onRestoreInstanceState(state);
            return;
        }

        SavedState ss = (SavedState)state;
        super.onRestoreInstanceState(ss.getSuperState());

        if (mAdapter != null) {
            mAdapter.restoreState(ss.adapterState, ss.loader);
            setCurrentItemInternal(ss.position, false, true);
        } else {
            mRestoredCurItem = ss.position;
            mRestoredAdapterState = ss.adapterState;
            mRestoredClassLoader = ss.loader;
        }
    }


### 2、FragmentPagerAdapter／FragmentStatePagerAdapter差别    

差别主要体现在左右页面切换时对`mCurItem-mOffscreenPageLimit和mCurItem＋mOffscreenPageLimit范围外`的页面的处理上。

（1）`fragments对象的处理`：FragmentPagerAdapter范围外fragments会保存在内存中(detach)，虽然可能fragment对应的View会被销毁；FragmentStatePagerAdapter范围外fragments不会保存在内存中(remove)。    
（2）`状态的处理`：FragmentPagerAdapter范围外fragments对应的SavedState会保存；FragmentStatePagerAdapter范围外fragments对应的SavedState不会保存。这个SavedState在Fragment的生命周期回调中供外部传参数，和Activity类似。    
（3）`适用场景`：相同数量的fragments，FragmentPagerAdapter内存较大，但页面切换更友好；FragmentStatePagerAdapter内存占用少，页面切换稍差。因此FragmentPagerAdapter适用于Fragment数量少的情况，FragmentStatePagerAdapter适用于Fragment数量多的情况。    


关于fragments对象的处理：
    
    //FragmentPagerAdapter
    @Override
    public Object instantiateItem(ViewGroup container, int position) {
        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }

        final long itemId = getItemId(position);

        // Do we already have this fragment?
        String name = makeFragmentName(container.getId(), itemId);
        Fragment fragment = mFragmentManager.findFragmentByTag(name);
        if (fragment != null) {
            if (DEBUG) Log.v(TAG, "Attaching item #" + itemId + ": f=" + fragment);
            mCurTransaction.attach(fragment);
        } else {
            fragment = getItem(position);
            if (DEBUG) Log.v(TAG, "Adding item #" + itemId + ": f=" + fragment);
            mCurTransaction.add(container.getId(), fragment,
                    makeFragmentName(container.getId(), itemId));
        }
        if (fragment != mCurrentPrimaryItem) {
            fragment.setMenuVisibility(false);
            fragment.setUserVisibleHint(false);
        }

        return fragment;
    }

    @Override
    public void destroyItem(ViewGroup container, int position, Object object) {
        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }
        if (DEBUG) Log.v(TAG, "Detaching item #" + getItemId(position) + ": f=" + object
                + " v=" + ((Fragment)object).getView());
        mCurTransaction.detach((Fragment)object);
    }


    //FragmentStatePagerAdapter
    @Override
    public Object instantiateItem(ViewGroup container, int position) {
        // If we already have this item instantiated, there is nothing
        // to do.  This can happen when we are restoring the entire pager
        // from its saved state, where the fragment manager has already
        // taken care of restoring the fragments we previously had instantiated.
        if (mFragments.size() > position) {
            Fragment f = mFragments.get(position);
            if (f != null) {
                return f;
            }
        }

        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }

        Fragment fragment = getItem(position);
        if (DEBUG) Log.v(TAG, "Adding item #" + position + ": f=" + fragment);
        if (mSavedState.size() > position) {
            Fragment.SavedState fss = mSavedState.get(position);
            if (fss != null) {
                fragment.setInitialSavedState(fss);
            }
        }
        while (mFragments.size() <= position) {
            mFragments.add(null);
        }
        fragment.setMenuVisibility(false);
        fragment.setUserVisibleHint(false);
        mFragments.set(position, fragment);
        mCurTransaction.add(container.getId(), fragment);

        return fragment;
    }

    @Override
    public void destroyItem(ViewGroup container, int position, Object object) {
        Fragment fragment = (Fragment)object;

        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }
        if (DEBUG) Log.v(TAG, "Removing item #" + position + ": f=" + object
                + " v=" + ((Fragment)object).getView());
        while (mSavedState.size() <= position) {
            mSavedState.add(null);
        }
        mSavedState.set(position, mFragmentManager.saveFragmentInstanceState(fragment));
        mFragments.set(position, null);

        mCurTransaction.remove(fragment);
    }

关于状态保存和恢复：

    //FragmentPagerAdapter
    @Override
    public Parcelable saveState() {
        return null;
    }

    @Override
    public void restoreState(Parcelable state, ClassLoader loader) {
    }


    //FragmentStatePagerAdapter
    @Override
    public Parcelable saveState() {
        Bundle state = null;
        if (mSavedState.size() > 0) {
            state = new Bundle();
            Fragment.SavedState[] fss = new Fragment.SavedState[mSavedState.size()];
            mSavedState.toArray(fss);
            state.putParcelableArray("states", fss);
        }
        for (int i=0; i<mFragments.size(); i++) {
            Fragment f = mFragments.get(i);
            if (f != null && f.isAdded()) {
                if (state == null) {
                    state = new Bundle();
                }
                String key = "f" + i;
                mFragmentManager.putFragment(state, key, f);
            }
        }
        return state;
    }

    @Override
    public void restoreState(Parcelable state, ClassLoader loader) {
        if (state != null) {
            Bundle bundle = (Bundle)state;
            bundle.setClassLoader(loader);
            Parcelable[] fss = bundle.getParcelableArray("states");
            mSavedState.clear();
            mFragments.clear();
            if (fss != null) {
                for (int i=0; i<fss.length; i++) {
                    mSavedState.add((Fragment.SavedState)fss[i]);
                }
            }
            Iterable<String> keys = bundle.keySet();
            for (String key: keys) {
                if (key.startsWith("f")) {
                    int index = Integer.parseInt(key.substring(1));
                    Fragment f = mFragmentManager.getFragment(bundle, key);
                    if (f != null) {
                        while (mFragments.size() <= index) {
                            mFragments.add(null);
                        }
                        f.setMenuVisibility(false);
                        mFragments.set(index, f);
                    } else {
                        Log.w(TAG, "Bad fragment at key " + key);
                    }
                }
            }
        }
    }

### 3、参考文档    

（1）[FragmentPagerAdapter和FragmentStatePagerAdapter源码中的三宝](https://segmentfault.com/a/1190000012455727)


------------------------------------------我是分割线--------------------------------------------


一、FragmentPagerAdapter

适合于 `Fragment`数量不多的情况。当某个页面不可见时，该页面对应的View可能会被销毁，但是所有的Fragment都会一直存在于内存中。如果Fragment需要保存的状态较多时，会导致占用内存较大，因此对于Fragment数量较多的情况，建议使用FragmentStatePagerAdapter。  



当使用FragmentPagerAdapter时，`ViewPager`必须有有效id。

    @Override
    public Object instantiateItem(ViewGroup container, int position) {
        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }

        final long itemId = getItemId(position);

        // Do we already have this fragment?
        String name = makeFragmentName(container.getId(), itemId);
        Fragment fragment = mFragmentManager.findFragmentByTag(name);
        if (fragment != null) {
            if (DEBUG) Log.v(TAG, "Attaching item #" + itemId + ": f=" + fragment);
            mCurTransaction.attach(fragment);
        } else {
            fragment = getItem(position);
            if (DEBUG) Log.v(TAG, "Adding item #" + itemId + ": f=" + fragment);
            mCurTransaction.add(container.getId(), fragment,
                    makeFragmentName(container.getId(), itemId));
        }
        if (fragment != mCurrentPrimaryItem) {
            fragment.setMenuVisibility(false);
            fragment.setUserVisibleHint(false);
        }

        return fragment;
    }

    @Override
    public void destroyItem(ViewGroup container, int position, Object object) {
        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }
        if (DEBUG) Log.v(TAG, "Detaching item #" + getItemId(position) + ": f=" + object
                + " v=" + ((Fragment)object).getView());
        mCurTransaction.detach((Fragment)object);
    }

    private static String makeFragmentName(int viewId, long id) {
        return "android:switcher:" + viewId + ":" + id;
    }

通过源码，可以看出FragmentPagerAdapter填充每一个页面的流程：  
（1）根据containerId和position得到fragment的tag  
（2）FragmentManager根据tag查找对应的Fragment，若存在则直接attach  
（3）若不存在，则调用getItem(position)，并设置tag，调用add方法添加

结论：  
（1） FragmentPagerAdapter中Fragment的tag 为"android:switcher:" + viewId + ":" + id;  
（2） getItem(position)中new Fragment()，并且不需要设置 tag，Fragment.instantiate()即可

二、FragmentStatePagerAdapter

适合于Fragment数量较多的情况。当页面不可见时， 对应的Fragment实例可能会被销毁，但是Fragment的状态会被保存。因此每个Fragment占用的内存会更少，但是页面切换会引起较大开销。  
当使用FragmentStatePagerAdapter时，ViewPager必须有有效id。

    @Override
    public Object instantiateItem(ViewGroup container, int position) {
        // If we already have this item instantiated, there is nothing
        // to do.  This can happen when we are restoring the entire pager
        // from its saved state, where the fragment manager has already
        // taken care of restoring the fragments we previously had instantiated.
        if (mFragments.size() > position) {
            Fragment f = mFragments.get(position);
            if (f != null) {
                return f;
            }
        }

        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }

        Fragment fragment = getItem(position);
        if (DEBUG) Log.v(TAG, "Adding item #" + position + ": f=" + fragment);
        if (mSavedState.size() > position) {
            Fragment.SavedState fss = mSavedState.get(position);
            if (fss != null) {
                fragment.setInitialSavedState(fss);
            }
        }
        while (mFragments.size() <= position) {
            mFragments.add(null);
        }
        fragment.setMenuVisibility(false);
        fragment.setUserVisibleHint(false);
        mFragments.set(position, fragment);
        mCurTransaction.add(container.getId(), fragment);

        return fragment;
    }

    @Override
    public void destroyItem(ViewGroup container, int position, Object object) {
        Fragment fragment = (Fragment)object;

        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }
        if (DEBUG) Log.v(TAG, "Removing item #" + position + ": f=" + object
                + " v=" + ((Fragment)object).getView());
        while (mSavedState.size() <= position) {
            mSavedState.add(null);
        }
        mSavedState.set(position, mFragmentManager.saveFragmentInstanceState(fragment));
        mFragments.set(position, null);

        mCurTransaction.remove(fragment);
    }

根据源码，FragmentStatePagerAdapter填充每一个页面的流程：  
（1）先试着找mFragments的list中index为position是否有Fragment，若有则直接返回；  
（2）若没有则调用getItem(position)实例化Fragment并设置初始状态，调用add    
 销毁流程：  
（1）保存 Fragment对应的状态
（2）remove Fragment

结论：  
（1）Fragment没有指定格式的tag  
（2）FragmentStatePagerAdapter会保持和恢复 Fragment的状态
