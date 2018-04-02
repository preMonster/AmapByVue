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

<style>

    #container{
        width:500px;
        height:400px;
        margin:0px auto;
    }

    .absolute-pull{
        position:absolute;
        z-index:999;
    }

    .pull-down-input{
        width:100px;
        height:40px;
        line-height:40px;
        font-size:12px;
        color:gray;
        border:1px #ccc solid;
        overflow: hidden;
        text-overflow:ellipsis;
        white-space: nowrap;
    }

    .pull-down-total-div{
        display:flex;
    }

    .pull-down-childs-div{
        width:max-content;
        overflow:auto;
    }

    .pull-down-child{
        width:100px;
        height:40px;
        line-height:40px;
        font-size:12px;
        color:#333;
        border:1px #eee solid;
        overflow: hidden;
        text-overflow:ellipsis;
        white-space: nowrap;
    }

    .box-shadow{
        box-shadow:0 6px 12px rgba(0,0,0,.175);
    }

    .max-height300{
        height:max-content;
        max-height:300px;
    }

    .pull-down-childs-div::-webkit-scrollbar {/*滚动条整体样式*/
            width: 0px;     /*高宽分别对应横竖滚动条的尺寸*/
            height: 0px;
    }

</style>