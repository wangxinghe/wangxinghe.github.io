---
layout: post
comments: true
title: "FragmentPagerAdapter和FragmentStatePagerAdapter分析"
description: "Fragment相关"
category: android
tags: [Android]
---
一、FragmentPagerAdapter

适合于 `Fragment`数量不多的情况。当某个页面不可见时，该页面对应的View可能会被销毁，但是所有的Fragment都会一直存在于内存中。如果Fragment需要保存的状态较多时，会导致占用内存较大，因此对于Fragment数量较多的情况，建议使用FragmentStatePagerAdapter。  

<!--more-->

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
