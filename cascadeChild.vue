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