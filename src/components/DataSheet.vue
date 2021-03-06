<template>
    <div id='datasheet'>
        <div id='datasheet-overlay' @click='destroyDataSheet'></div>
        <div id="datasheet-exit-button" @click='destroyDataSheet'><div id="datasheet-exit-button-text">&#x2573;</div></div>
        <div id='datasheet-container'>
            <div id="back-arrow" @click="goToPrevious()">
                <span v-tooltip.left="{
                content: (history.length > 1) ? history[history.length - 1][1] : 'Exit',
                class: 'back-arrow-tooltip'}">
                &#x1f844;</span>
            </div>
            <transition name='fade' mode='out-in'>
                <data-sheet-basic v-if="dataReceived" :key="id" v-bind:jsonElementData="jsonElementData"></data-sheet-basic>
                <data-sheet-temporal-edge v-else-if="edgeDataReceived" v-bind:jsonElementData="edgeData" v-bind:stratotypeData="stratotypeData"></data-sheet-temporal-edge>
                <div id="loading-icon" v-else>
                    <img src="../assets/loading.svg" width="120px" height="120px">
                    <p>Loading...</p>
                </div>
            </transition>
        </div>
    </div>
</template>

<script>
import EventBus from "../assets/event-bus.js"
import DataSheetBasic from './DataSheetBasic.vue'
import DataSheetTemporalEdge from './DataSheetTemporalEdge.vue'
export default {
    name: 'DataSheet',
    components: {
        DataSheetBasic,
        DataSheetTemporalEdge
    },
    props: {
        id: String,
    },
    data (){
        return {
            history: [[this.id, 'Exit']],
            lastNode: false,
            jsonElementData: null,
            dataReceived: false,
            edgeData: null,
            stratotypeData: [],
            edgeDataReceived: false
        }
    },
    methods: {
        httpRequestAsync: function(url, dataType){
            var this_ = this
            var xmlHttp = new XMLHttpRequest()
            xmlHttp.onreadystatechange = function () {
                if (xmlHttp.readyState == 4 && xmlHttp.status == 200){
                    var data = xmlHttp.responseText
                    if (dataType == 'jsonElementData'){
                        this_.jsonElementData = JSON.parse(data)
                        this_.dataReceived = true
                    } else if (dataType == 'edgeData'){
                        this_.edgeData = JSON.parse(data)
                        this_.dataReceived = false
                        if (this_.edgeData.result.primaryTopic.stratotype != null && !(this_.edgeData.result.primaryTopic.stratotype instanceof Array)){
                            this_.httpRequestAsync('https://vocabs.ands.org.au/repository/api/lda/csiro/international-chronostratigraphic-chart/2018-revised-corrected/resource.json?uri=' + this_.edgeData.result.primaryTopic.stratotype._about, 'stratotypeData')
                        } else if (this_.edgeData.result.primaryTopic.stratotype != null && (this_.edgeData.result.primaryTopic.stratotype instanceof Array)){
                            for (var stratotype in this_.edgeData.result.primaryTopic.stratotype){
                                this_.httpRequestAsync('https://vocabs.ands.org.au/repository/api/lda/csiro/international-chronostratigraphic-chart/2018-revised-corrected/resource.json?uri=' + this_.edgeData.result.primaryTopic.stratotype[stratotype]._about, 'stratotypeData')
                            }
                        }
                        this_.edgeDataReceived = true
                    } else if (dataType == "stratotypeData"){
                        this_.stratotypeData.push(JSON.parse(data))
                    }
            }
        }
        xmlHttp.open("GET", url, true)
        xmlHttp.send(null)
        },
        destroyDataSheet: function() {
            EventBus.$emit('destroy-data-sheet', null)
        },
        goToPrevious: function() {
            if (this.edgeDataReceived){
                this.edgeDataReceived = false
                this.dataReceived = true
                this.stratotypeData = []
            }else{
                if (this.history.length == 1) {
                    this.destroyDataSheet()
                } else {
                    this.dataReceived = false
                    var previousNode = this.history.pop()
                    var url = this.history[this.history.length - 1][0]
                    var requestURL = 'https://vocabs.ands.org.au/repository/api/lda/csiro/international-chronostratigraphic-chart/2018-revised-corrected/resource.json?uri=' + url
                    this.$emit('input', url)
                    this.httpRequestAsync(requestURL, 'jsonElementData')
                }
            }
        }
    },
    mounted(){
        var requestURL = 'https://vocabs.ands.org.au/repository/api/lda/csiro/international-chronostratigraphic-chart/2018-revised-corrected/resource.json?uri=' + this.id
        this.httpRequestAsync(requestURL, 'jsonElementData')
        EventBus.$on('get-more-data', url => {
            this.history.push([this.id, this.jsonElementData.result.primaryTopic.label._value])
            this.httpRequestAsync('https://vocabs.ands.org.au/repository/api/lda/csiro/international-chronostratigraphic-chart/2018-revised-corrected/resource.json?uri=' + url, 'edgeData')
        })
        EventBus.$on('go-back', payload => {
            this.edgeDataReceived = !payload
            this.dataReceived = payload
            this.stratotypeData = [];
        }),
        EventBus.$on('update-data', node =>{
            this.dataReceived = false
            requestURL = 'https://vocabs.ands.org.au/repository/api/lda/csiro/international-chronostratigraphic-chart/2018-revised-corrected/resource.json?uri=' + node[0]
            this.$emit('input', node[0])
            this.httpRequestAsync(requestURL, 'jsonElementData')
            this.history.push(node)
    })
    }
}
</script>


<style>
    p{
        text-align: center;
    }
    h3 {
        margin: 40px 0 0;
    }
    ul {
        list-style-type: none;
        padding: 0;
    }
    li {
        display: inline-block;
        margin: 0 10px;
    }
    a {
        color: #42b983;
    }
    #datasheet {
        position: absolute;
        height: 100%;
        width: 100%;
        top: 0;
        bottom: 0;
    }
    #datasheet-overlay{
        overflow: visible;
        position: fixed;
        top: 0;
        height: 100%;
        width: 100%;
        opacity: 0.75;
        background-color: black;
        z-index: 1;
    }
    #datasheet-container{
        position: fixed;
        overflow: auto;
        width: 50%;
        height: 80%; /* Change this to 'max-height' to get a dynamic container*/
        margin-top: 5%;
        margin-bottom: 5%;
        left: 25%;
        right: 25%;
        z-index: 2;
        box-shadow: 2px 2px 5px black;
        border: 1px solid black;
        padding: 20px;
        background-color: #E9E9E9;
    }
    #datasheet-exit-button{
        transition: opacity .5s;
        position: fixed;
        top: 0;
        right: 0;
        height: 75px;
        width: 75px;
        padding-top: 0px;
        background-color: black;
        opacity: 0.25;
        border-bottom-left-radius: 95%;
        z-index: 2;
        -webkit-user-select: none;
        -moz-user-select: none;
        -ms-user-select: none;
        user-select: none;   
    }
    #datasheet-exit-button:hover{
        opacity: 0.75;
        transition: opacity .5s;
        cursor: pointer;
    }
    #datasheet-exit-button-text{
        position: fixed;
        font-size: 26px;
        font-weight: bold;
        color: white;
        top: 10px;
        right: 20px;
        z-index: 3;
    }
    #loading-icon{
        margin: auto;
        position: absolute;
        top: 25%;
        left: 0;  
        right: 0;
        -webkit-user-select: none;
        -moz-user-select: none;
        -ms-user-select: none;
        user-select: none;   
    }
    .vue-tooltip.tooltip{
        opacity: 0.75;
    }
    .tooltip-text{
        border-bottom: 1px dotted #2C3E50;
    }
    .tooltip-text:hover{
        cursor: help;
    }
#back-arrow{
  font-size: 68px;
  position: fixed;
  transition: opacity .5s;
  opacity: 0.25;
  -webkit-user-select: none;
  -moz-user-select: none;  
  -ms-user-select: none;  
  user-select: none;   
}

#back-arrow:hover{
  color: #0C415A;
  opacity: 1;
  cursor: pointer;
}
.vue-tooltip.back-arrow-tooltip{
 background-color: black;
 color: white;
 font-size: 24px;
}
.vue-tooltip.back-arrow-tooltip .tooltip-arrow{
 border: none;
}

</style>
