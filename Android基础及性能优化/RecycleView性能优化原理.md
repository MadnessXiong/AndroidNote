### RecycleView的性能优化可以从以下几方面展开

- **缓存：**

  - Scrap:缓存的是屏幕内正在显示的view。如屏幕每隔16ms刷新一次，那么就是用的它的数据。
  - Cache:缓存的是刚出屏幕的view。它不会被回收，数据不会改变。复用时不会经过重新绑定的过程。不会调用onBindViewHolder(),默认大小为2

  > ListView的条目滑出屏幕后数据就会变成脏数据，如果这时再滑回来，虽然还是一样的条目，但是其实是已经调用过了getView(),进行了重新赋值。而RecycleView的Cache则直接缓存的了刚滑出的条目，如果这时再滑回来，那么还是刚才的条目，并没有调用onBindViewHolder()重新赋值，节省了一部分性能。

  - ViewCacheExtension：自定义缓存，使用时直接返回一个ItemView，适用于条目较少，及固定条目的列表。这样每次更新时都是返回固定的数据，不用重复重新创建及绑定。节省了一部分性能。直接通过recyclerView.setViewCacheExtension(RecyclerView.ViewCacheExtension),设置即可。

  - RecycledViewPool：这个缓存里的数据都是脏的，也就是复用时需要进行重新绑定,会调用onBindViewHolder(),相当于ListVIew的ConvertView。同时多个RecycleView是可以共用一个RecycledViewPool的，以减少总缓存的数量，节省性能。代码如下：

    ```java
            recyclerView.setRecycledViewPool(recycledViewPool);
            recyclerView1.setRecycledViewPool(recycledViewPool);
            recyclerView2.setRecycledViewPool(recycledViewPool);
    ```

- **setInitialPrefetchItemCount():** 这个方法主要用来预加载，当2个RecycleView嵌套时，如一个竖向RecycleView里嵌套一个横向的RecycleView，这时横向的RecycleView可以设置setInitialPrefetchItemCount()，对数据进行预加载，提高效率。此方法只有被嵌套的RecycleView里设置时才有效，同时必须是LinearLayoutManager来设置，其他无效。

  > Android 5.0之后，RecycleView引入了RenderThread线程，将主线程用于ui渲染的操作放在了此线程，此时空闲的主线程就可以进行其他操作，如RecycleView的预加载等。

- **setHasFixedSize(true):** 如果adapter的数据变化不会导致RecycleView的大小变化，那么调用这个方法后，刷新RecycleView不会进执行完整的绘制流程，只会进行layout()过程，以节省性能。需要注意的是不能使用notifyDataSetChanged()刷新，此方法会调用requestLayout()，会导致整个布局进行重绘。

- **DiffUtil:**页面刷新时，调用notifyDataSetChanged会导致整个布局重新绘制，重新绑定所有viewHolder，而且可能会失去动画效果。如删除一个条目时的动画。那么DiffUtill，就适用于整个页面需要刷新，但是有部分数据可能相同的情况。代码如下：

  ```
         		//DiffUtil.Callback主要是提供给系统计算diff
  					new DiffUtil.Callback(){
          		//返回原数据集合大小
              @Override
              public int getOldListSize() {
                  return 0;
              }
  						//返回现数据集合大小	
              @Override
              public int getNewListSize() {
                  return 0;
              }
  						//判断是否是同一个item，可以通过position取出2个集合里的数据根据id或其他属性进行比较
              @Override
              public boolean areItemsTheSame(int oldItemPosition, int newItemPosition) {
                  return false;
              }
  						//如是相同的item，内容是否相同
              @Override
              public boolean areContentsTheSame(int oldItemPosition, int newItemPosition) {
                  return false;
              }
            	//如果是同一个条目，并且内容不同，那么回调此方法，如果要对整个条目进行全量更新，那么可以不复写此方法，如果只是改某个或某几个字段，那么可以选择复写此方法。减少修改的部分，提升性能。
              @Nullable
              @Override
              public Object getChangePayload(int oldItemPosition, int newItemPosition) {
                //在此通过position取出数据，对新老数据进行比较，找出不同的部分，然后存在一个bundle或其他对象中返回
                  return ;
              }
          };
  ```

  设置完CallBack后，再就是在Adapter里使用了：

  ```
      @Override
      public void onBindViewHolder(@NonNull RecyclerView.ViewHolder holder, int position, @NonNull List payloads) {
          super.onBindViewHolder(holder, position, payloads);
        			//通过payloads来判断是增量还是全量更新，这个payloads就是getChangePayload返回的数据
               if (payloads.isEmpty()){
                 //如果是全量更新，则调用默认的2个参数的onBindViewHolder()，完成列表更新
             	 onBindViewHolder(holder,position)
         			 }else {
              	//如果是增量更新，这里则完成增量更新的部分
          		}
      }
  ```

  一般RecycleView的Adapter默认使用2个参数的onBindViewHolder(),如果要用到DiffUtil，则要使用3个参数的onBindViewHolder()，最后就是刷新数据：

  ```
  //这里的callback，就是上面设置的callback,第二个参数代表是否有列表移动，一般写false
  DiffUtil.calculateDiff(Callback cb, boolean detectMoves).dispatchUpdatesTo(adapter);
  ```

  以上是DiffUtil的使用。同时DiffUtil对diff的计算是需要耗费时间的，如果列表很大时，可能会消耗比较多的时间，可以考虑将计算工作放到子线程中去执行。

### 附：

- 前面说过RecycleView的缓存有可能不经过onBindViewHolder(),如Cache，ViewCacheExtension。那么此时一些操作在onBindViewHolder()里做就不合适，比如统计数据等。RecycleView提供了另外一个方法onViewAttachedToWindow(),此方法保证，每一个item在显示到屏幕上时都会被调用。
