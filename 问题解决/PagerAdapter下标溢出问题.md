## 开发环境
* Android Studio 2.3
* JDK 1.8
* gradle 2.2.3
* buildToolsVersion 25.0.1

## 问题表现
* 组件:Viewpager+PagerAdapter,View是Fragment。
* 错误Log，下标溢出
* 定位代码错误地址:更新Adapter数据，导致移除错误

```
container.removeView(fragments.get(position).getView()); // 移出viewpager两边之外的page布局
```
* 解决办法，数组下标不一致增加判断

```
if(fragments != null && position < fragments.size()){
    container.removeView(fragments.get(position).getView()); // 移出viewpager两边之外的page布局
}
```
* 增加判断代码之后问题还是出现下标溢出，再次定位代码报错地址。

```
public void onPageSelected(int i) {
       fragments.get(currentPageIndex).onPause(); // 调用切换前Fargment的onPause()
       //        fragments.get(currentPageIndex).onStop(); // 调用切换前Fargment的onStop()
       if (fragments.get(i).isAdded()) {
           //            fragments.get(i).onStart(); // 调用切换后Fargment的onStart()
           fragments.get(i).onResume(); // 调用切换后Fargment的onResume()
       }
       currentPageIndex = i;

       if (null != onExtraPageChangeListener) { // 如果设置了额外功能接口
           onExtraPageChangeListener.onExtraPageSelected(i);
       }

   }
```
* currentPageIndex参数跟onPageSelected方法中的i变量对不上
* 增加Fragment组件数量需要重置当前指示器的下标，修改下标代码如下:

```
public void setPagerItems(List<BaseFragment> items) {
       if (items != null) {
           for (int i = 0; i < fragments.size(); i++) {
               fragmentManager.beginTransaction().remove(fragments.get(i)).commit();
           }
           this.fragments = items;
           currentPageIndex = 0;//指示器重置
           notifyDataSetChanged();
       }
   }
```

## 问题总结
* FragmentManager的管理机制不够熟悉
* Java变量控制有问题
* 解决问题思路不够清晰

#### 感谢旺哥指导以及一起加班的同事。
