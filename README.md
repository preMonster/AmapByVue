# vue+高德地图原生制作五级城市级联框（并加入高德地图区域渲染）

最近在学vue,因为需要一个关于城市的级联框，又不想用插件，所以就自己做了一个，效果自己用着还行，大家感兴趣的可以看看。先上图：
![这里写图片描述](https://img-blog.csdn.net/20180402105939240?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3ByZU1vbnN0ZXI=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

用到的技术：
> 
> - **vue**
> - **高德地图简单API应用**
> - **原生HTML/JS/CSS**

-------------------

 

一. 思路分析
-------
想要实现一个上图的效果，实际上我们可以拆分成左边的级联搜索框和右边的高德地图渲染。级联框的数据来源来自高德地图的API,后面我会提到地图数据的引用。

左边的级联框我们可以分成一级为一个组件，一个组件包含该层次所有的区域选择。当我选择一个区域，他会去调用API，获得所选区域子级区域。这里有两种思路：
>1. 一次性加载出所有地图数据
>2. 当点击选择区域时，去访问加载它的对应子级。
	
第一种情况第一次渲染时间过长，但加载完成后交互性好，用户体验高。第二种思路每次渲染时间相差不大，但是点击时渲染效果用户体验一般。

综合两者优缺点，我选择了第二种思路，防止用户加载数据过长。

二.vue组件实现
---------

 **1. 顶层组件**
 
顶层组件里面主要包含了地图渲染，级联框等最上层组件。先上代码：

```
<template>
    <div>
        <!-- 下拉选择框 -->
        <div @click="changeCascadeState" class="pull-down-input" :title="street">
            <span>{{street}}</span>
            <Icon :type="theOpenState?'ios-arrow-up':'ios-arrow-down'"></Icon>
        </div>
        <!-- 级联搜索框 -->
        <div class="pull-down-total-div absolute-pull" v-if="theOpenState">
            <div ref="childsDiv" class="pull-down-childs-div max-height300 box-shadow">
                <div v-for="(item,index) of data" class="pull-down-child" @click="changeCascadeChildState(index)" :title="item.name">
                    <span>{{item.name}}</span>
                    <Icon :type="item.districtList.length>0?'ios-arrow-right':''"></Icon>
                </div>
            </div>
            <!-- 子组件级联搜索框 -->
            <div v-if="theOpenChild" class="pull-down-childs-div" :style="cascadeChildDivStyle">
                <casChild :data="data[chooseIndex].districtList" levels="1"
                          :streets="streets" :setStreet="setStreet" :map="map"
                          :polygonMap="polygonMap"></casChild>
            </div>
        </div>
        <!-- 地图显示 -->
        <div>
            <div id="container"></div>
        </div>
    </div>
</template>
```
这里是父组件的相关html代码。逻辑代码如下：

```
<script>
    import casChild from './cascadeChild.vue'
    import AMap from 'AMap';   //在页面中引入高德地图

    export default{
        name:"aMap",
        data:function(){
            return {
                comName:"中国",//最高一级，默认中国
                data:[],//加载子组件所需要的数据数组
                street:"街道选择框",//下拉选择框显示值
                streets:["中国"],//级联数组
                theOpenState:false,//下拉选择框打开状态
                theOpenChild:false,//子组件选择框打开状态
                cascadeChildDivStyle:{//子组件上顶部平行样式
                    'margin-top':'10px'
                },
                chooseIndex:-1,//选择下标
                map:{},//高德地图
                polygon:{}//高德地图区域渲染
            }
        },
        components:{
            'casChild':casChild
        },
        mounted:function(){
            this.loadmap(this.comName);     //加载地图和相关组件
        },
        methods:{
            loadmap(name){//初始化加载地图相关组件
                this.map = new AMap.Map('container', {//地图初始化
                    zoom: 12
                });
                let districtSearch;//查找地图插件
                let that = this;
                this.data = [];
                AMap.service('AMap.DistrictSearch',function(){//回调函数
                    //实例化DistrictSearch
                    districtSearch = new AMap.DistrictSearch({
                        level : 'country',  
                        subdistrict : 1    
                    });
                    //TODO: 使用districtSearch对象调用行政区查询的功能
                        districtSearch.setSubdistrict(1);
                        //TODO: 使用districtSearch对象调用行政区查询的功能
                        //调用查询方法
                        districtSearch.search(name,function(status, result){
                            //TODO : 按照自己需求处理查询结果
                            that.data = result.districtList?result.districtList:[];
                        });
                });

                this.polygon = new AMap.Polygon({  //行政区边界渲染，使用多边形覆盖物实现，初始化this.polygon状态
                    map: this.map,
                    strokeWeight: 1,
                    fillOpacity: 0.7,
                    fillColor: '#CCF3FF',
                    strokeColor: '#CC66CC'
                });
            },
            changeCascadeState(){//改变选择下拉框的状态
                this.theOpenState = !this.theOpenState;
                if(!this.theOpenState){//当关闭选择下拉框的时候，将子组件状态也设为关闭
                    this.theOpenChild = false;
                }
            },
            changeCascadeChildState(index){//改变子级选择框的组件
                this.theOpenChild = !this.theOpenChild;
                this.chooseIndex = index;//选择父组件的值
                this.cascadeChildDivStyle['margin-top'] = index*40-this.$refs.childsDiv.scrollTop+"px";//定义子组件的高度，因为子组件展开的上顶部要与选择框平行
            },
            setStreet(){//返回最后选取的街道信息，并将下拉选择框的值改变为“.../.../...”格式
                let s = "";
                // for(var i in this.streets){//遍历数组，返回以
                //     if(this.streets[i]){
                //         s+=this.streets[i];
                //         if(i<this.streets.length-1&&this.streets[parseInt(i)+1]!=null){
                //             s+="/";
                //         }
                //     }
                // }
                s = this.streets.filter(function(s){
                    return s!=null;//过滤null
                }).join('/');
                this.street = s;
                this.changeCascadeState(-1);
            },
            polygonMap(result){//渲染地图
                this.polygon.setMap(null);// 清空地图渲染，方便重新绘制地图
                this.polygon = new AMap.Polygon({ // 行政区边界渲染，使用多边形覆盖物实现
                    map: this.map,
                    strokeWeight: 1,
                    fillOpacity: 0.7,
                    fillColor: '#CCF3FF',
                    strokeColor: '#CC66CC'
                });
                this.polygon.setPath(result.boundaries); // 渲染地图通过查询地区所得到的数组
                if(result.level==="street"){
                    this.map.setZoom(14); // 当最后一级为街道，将地图放大到14
                }
                this.map.setFitView();// 地图自定义
                this.map.setCenter(result.center);// 设置地图中心区域
            }
        }
    }    
</script>
```

 **2. 子组件**
 
子组件采用了vue的组件递归调用：
  
>注意：组件在它的模板内可以递归地调用自己。不过，只有当它有 name 选项时才可以这么做。

子组件包含了各个子级级联框的渲染，选择和地图区域绘制。

页面代码：

```
<template>
    <div class="pull-down-total-div">
        <!-- 本身选择按钮集合 -->
        <div ref="childsDiv" class="pull-down-childs-div max-height300 box-shadow">
            <div v-for="(item,index) of data" class="pull-down-child" @click="changeCascadeChildState(index)" :title="item.name">
                <span>{{item.name}}</span>
                <Icon :type="levels!='4'?'ios-arrow-right':''"></Icon>
            </div>
        </div>
        <!-- 子级组件（递归调用） -->
        <div v-if="theOpenChild" class="pull-down-childs-div" :style="cascadeChildDivStyle">
            <casChild :data="childData.length>0?childData[0].districtList:[]" :levels="parseInt(levels)+1+''"
                      :streets="streets" :setStreet="setPropsStreet" :map="map"
                      :polygonMap="polygonMap" :theOpenParentState="chooseIndexChangeState"
                      :changeParentChooseIndexState="changeChooseIndexState"></casChild>
        </div>
    </div>
</template>
```
下面是逻辑代码部分：

```
<script>
    import AMap from 'AMap';   //在页面中引入高德地图

    export default{
        name: 'casChild',
        props: ['data','levels','streets','setStreet','map',
                'polygonMap','theOpenParentState','changeParentChooseIndexState'],
        data:function(){
            return {
                childData:[],//子组件所需要的数据数组
                theOpenChild:false,//打开子组件的状态
                cascadeChildDivStyle:{//子组件上顶部平行样式
                    'margin-top':'10px'
                },
                chooseIndex:-1,//选择下标
                chooseIndexChangeState:false//下标是否改变的状态
            }
        },
        watch:{
            theOpenParentState:function(val,oldval){//监听theOpenParentState（父组件的选择是否改变，如果改变，子组件的子组件状态设为false）
                this.theOpenChild = !val;
            }
        },
        methods:{
            loadmap(parentName){//渲染地图
                let districtSearch;
                let that = this;
                AMap.service('AMap.DistrictSearch',function(){//回调函数
                    //实例化DistrictSearch
                    districtSearch = new AMap.DistrictSearch({
                        extensions: 'all',//设为all,才能得到渲染地图所需要的bouldes
                        level : 'country',  
                        subdistrict : 1    
                    });
                    //TODO: 使用districtSearch对象调用行政区查询的功能
                        districtSearch.setSubdistrict(1);
                        //TODO: 使用districtSearch对象调用行政区查询的功能
                        //调用查询方法
                        districtSearch.search(parentName,function(status, result){
                            //TODO : 按照自己需求处理查询结果
                            that.childData = result.districtList?result.districtList:[];
                            if(!that.childData[0].districtList){//如果最后一级为街道，或者最后一级子组件没有数据了，则返回最终结果，并关闭下拉选择框
                                that.setPropsStreet();
                            }
                            that.polygonMap(result.districtList[0]);//根据坐标数组渲染区域
                        });
                });
            },
            changeCascadeChildState(index){//改变子组件状态
                this.changeStreets(this.data[index].name);//改变街道数组，将名字存入数组，其实也可以存入数组
                if(this.levels==='4'){//当最后一级为街道，渲染地图，并返回最终结果（等级说明：0——国家，1——省份，2——城市，3——区县，4——街道）
                    this.polygonMap(this.data[index]);
                    this.setPropsStreet();
                    return;
                }

                
                this.childData = [];

                if(this.theOpenParentState){//当父组件是下标改变选择状态时，点击选择，会将父组件下标改变状态重置
                    this.changeParentChooseIndexState();
                }

                if(index===this.chooseIndex||this.chooseIndex===-1){//当选择的下标跟上一个选择的下标一样则关闭子组件选择框，或者当第一次选择才可以
                    this.theOpenChild = !this.theOpenChild;
                }else{
                    this.changeChooseIndexState();//如果都不满足，则说明在改变下标
                }

                this.chooseIndex = index;//将下标设为选择值

                this.cascadeChildDivStyle['margin-top'] = index*40-this.$refs.childsDiv.scrollTop+"px";
                this.loadmap(this.data[index].name);//加载地图
            },
            changeStreets(name){//改变街道数组数据
                this.streets[this.levels] = name;
                for(let i=parseInt(this.levels)+1;i<this.streets.length;i++){//如果点击了父级组件以上的选择，则将点击以下的组件数据设为null
                    this.streets[i] = null;
                }
            },
            changeChooseIndexState(){//改变下标改变状态
                this.chooseIndexChangeState = !this.chooseIndexChangeState;
            },
            setPropsStreet(){//返回最终结果
                this.setStreet();
            }
        }
    }    
</script>
```
不停的递归调用本身，递归出口在当它没有子级数据或者它的等级为最低等级街道“street”。

 - `if(this.levels==='4')`
 - `if(!that.childData[0].districtList)`

在做的过程中，我遇到一个问题：当我生成层次达到3个层次以上时，当我选择爷爷级组件不同选择框，我想达成的效果是，父级组件更换数据，子组件关闭。但是在我达到的是，父组件更换数据，子组件未关闭。
后来找到了原因：因为每个组件都有一个`theOpenChild`控制子组件的状态显示，我爷爷组件最多控制自身的`theOpenChild`值，无法直接控制到父亲级别的组件的`theOpenChild`值（也就是孙子级别组件的显示）。

解决办法：自定义一个`chooseIndexChangeState`的值，负责监听爷爷级选择是否改变，将该值传向子组件设为`theOpenParentState`。并对该值进行监听，如果改变了，就将父亲组件的`theOpenChild`进行改变。

主要代码：

```
   watch:{
            theOpenParentState:function(val,oldval){//监听theOpenParentState（父组件的选择是否改变，如果改变，子组件的子组件状态设为false）
                this.theOpenChild = !val;
            }
        }
```

三.高德地图Vue引用介绍
-------------
>你首先需要得到高德地图开发人员的授权,授权说明：http://lbs.amap.com/api/javascript-api/summary [ 高德地图key值获取 ]。

因为我的代码是在webpack环境下打包的，所以我就讲一下在webpack下vue引用高德地图。

 **1. webpack引入高德**
 在`webpack.base.conf.js`中找到`module.exports`对象。添加`externals: {  'AMap': 'AMap' }`代码。
 
**2. html引入高德js**
在index.html页面加入`<script type="text/javascript" src="http://webapi.amap.com/maps?v=1.4.4&key='你的key值'"></script>`

**3. 在页面中引入高德js**
在vue文件中import进来。`import AMap from 'AMap';   //在页面中引入高德地图`

**4. 地图生成**
地图初始化生成代码：

```
new AMap.Map('container', {//地图初始化
                    zoom: 12//地图缩放尺寸（还有很多属性，具体查阅API）
                });
```

**5. 地图区域渲染**
渲染代码：

```
 polygonMap(result){//渲染地图
                this.polygon.setMap(null);// 清空地图渲染，方便重新绘制地图
                this.polygon = new AMap.Polygon({ // 行政区边界渲染，使用多边形覆盖物实现
                    map: this.map,
                    strokeWeight: 1,
                    fillOpacity: 0.7,
                    fillColor: '#CCF3FF',
                    strokeColor: '#CC66CC'
                });
                this.polygon.setPath(result.boundaries); // 渲染地图通过查询地区所得到的数组
                if(result.level==="street"){
                    this.map.setZoom(14); // 当最后一级为街道，将地图放大到14
                }
                this.map.setFitView();// 地图自定义
                this.map.setCenter(result.center);// 设置地图中心区域
            }
```
这样我们就能简单生成map图像了。
