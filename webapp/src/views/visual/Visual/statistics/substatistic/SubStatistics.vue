/** Copyright 2020 Tianshu AI Platform. All Rights Reserved.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 * =============================================================
 */

<template>
  <div class="statistics-container">
    <div class="statistics-title" @click="showContent()">
      <div :class="[showFlag ? 'sub1' : 'sub']">
        <div class="my-label">
          <div class="my-text">
            <span>{{ categoryName }}</span>
          </div>
          <div class="circle-father"><div class="circle" /></div>
          <div class="triangle-father">
            <div class="triangle" />
          </div>
        </div>
      </div>
      <div class="line1" />
      <div :class="['line2', showFlag ? '' : 'linestyle']" />
    </div>
    <div :class="[showFlag ? '' : 'showClass']">
      <div>
        <div v-for="(oneRunData, runIdx) in runData" :key="runIdx" :class="['statistics-content']">
          <div
            v-for="(item, index) in oneRunData"
            v-show="getDataSetsState[item[0]]"
            :id="[idArray[item[4]]]"
            :key="index"
            class="allStatisticContainer"
          >
            <statisticContainer
              :data="item[2]"
              :ttlabel="item[0]"
              :tag="item[1]"
              :itemp="item[4]"
              :componentName="componentName"
              :runColor="getStatisticColor[item[3] % 5]"
              :divId="idArray[item[4]]"
              :parentComponent="parentComponent"
              checked="false"
              class="statisticContaierContent"
            />
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
<script>
import { createNamespacedHelpers } from 'vuex';
import { statisticContainer } from '../drawStatistic';

const {
  mapGetters: mapStatisticGetters,
  mapActions: mapStatisticActions,
  mapMutations: mapStatisticMutations,
} = createNamespacedHelpers('Visual/statistic');
const { mapState: mapLayoutStates } = createNamespacedHelpers('Visual/layout');
export default {
  components: {
    statisticContainer,
  },
  props: {
    categoryInfo: String,
  },
  data() {
    return {
      showFlag: '',
      componentName: this.categoryInfo,
      categoryName: '',
      allData: [],
      runData: [],
      idArray: [],
      parentComponent: true,
    };
  },
  computed: {
    ...mapStatisticGetters([
      'getInitStateFlag',
      'getBinNum',
      'getDistData',
      'getHistData',
      'getMode',
      'getShowStatisticFlag',
      'getDataSets',
      'getDataSetsState',
      'getStatisticColor',
      'getHistShow',
      'getDistShow',
    ]),
    ...mapLayoutStates(['userSelectRunFile']),
  },
  watch: {
    getMode(curMode) {
      if (this.categoryInfo === 'histogram') {
        if (curMode === '??????') {
          this.componentName = 'threed';
        } else {
          this.componentName = 'orthographic';
        }
      }
    },
    getBinNum() {
      if (this.categoryInfo === 'histogram') {
        this.manageHistData(true);
      }
    },
    getHistData(data) {
      if (this.categoryInfo === 'histogram') {
        this.allData = data;
        this.idArray = [];
        for (let i = 0; i < this.allData.length; i += 1) {
          this.idArray.push(`myHistogramScale${i}`);
        }
        this.setRunData();
      }
    },
    getDistData(data) {
      if (this.categoryInfo === 'distribution') {
        this.allData = data;
        this.idArray = [];
        for (let i = 0; i < this.allData.length; i += 1) {
          this.idArray.push(`myDistributionScale${i}`);
        }
        this.setRunData();
      }
    },
    userSelectRunFile() {
      this.setDatasetsShow();
    },
    getHistShow(val) {
      if (this.categoryInfo === 'histogram') {
        this.showFlag = val;
        if (val) {
          document.getElementsByClassName('statistics-container')[0].scrollIntoView(true);
        }
      }
    },
    getDistShow(val) {
      if (this.categoryInfo === 'distribution') {
        this.showFlag = val;
        if (val) {
          this.setDistData();
          document.getElementsByClassName('statistics-container')[1].scrollIntoView(true); // ???????????????????????????
        }
      }
    },
  },
  created() {
    if (this.categoryInfo === 'histogram') {
      this.categoryName = '?????????';
      this.showFlag = true;
      this.setHistShow(true); // ????????????
      this.setHistData();
    } else {
      this.categoryName = '?????????';
      this.showFlag = false;
      this.setDistShow(false);
    }
  },
  mounted() {
    window.addEventListener('scroll', this.handleScroll, true); // ???????????????
    this.setDatasetsShow();
  },
  methods: {
    ...mapStatisticActions(['featchAllDistData', 'featchAllHistData']),
    ...mapStatisticMutations([
      'manageHistData',
      'setDataSetsState',
      'setInitStateFlag',
      'setHistShow',
      'setDistShow',
    ]),
    showContent() {
      if (this.categoryInfo === 'histogram') {
        this.setHistShow(!this.showFlag);
      } else {
        this.setDistShow(!this.showFlag);
      }
    },
    setHistData() {
      if (this.getMode === '??????') {
        this.componentName = 'threed';
      } else {
        this.componentName = 'orthographic';
      }
      // ??????????????????????????????true??????????????????????????????
      if (this.getInitStateFlag) {
        this.setInitStateFlag(false);
        return;
      }
      if (!this.getInitStateFlag && this.getHistData.length === 0) {
        this.featchAllHistData();
        return;
      }
      // ?????????
      this.allData = this.getHistData;
      this.idArray = [];
      for (let i = 0; i < this.allData.length; i += 1) {
        this.idArray.push(`myHistogramScale${i}`);
      }
      this.setRunData();
    },
    setDistData() {
      this.componentName = 'overlook';
      if (this.getDistData.length === 0) {
        this.featchAllDistData();
        return;
      }
      this.allData = this.getDistData;
      this.idArray = [];
      for (let i = 0; i < this.allData.length; i += 1) {
        this.idArray.push(`myDistributionScale${i}`);
      }
      this.setRunData();
    },
    handleScroll() {
      // ???????????????????????????histDist?????????????????????????????????
      if (document.getElementsByClassName('statistics-container')[1] !== undefined) {
        const curDistHeight = document
          .getElementsByClassName('statistics-container')[1]
          .getBoundingClientRect().top;
        if (curDistHeight < window.innerHeight * 0.7) {
          this.setDistShow(true);
        } else if (curDistHeight > window.innerHeight) {
          this.setHistShow(true);
        }
      }
    },
    setDatasetsShow() {
      const stateTemp = [];
      for (let i = 0; i < this.getDataSets.length; i += 1) {
        stateTemp[this.getDataSets[i]] = false;
      }
      for (let i = 0; i < this.userSelectRunFile.length; i += 1) {
        stateTemp[this.userSelectRunFile[i]] = true;
      }
      // userselectRunFiles?????????????????????statistic??????run??????
      this.setDataSetsState(stateTemp);
    },
    setRunData() {
      // ???run?????????????????????????????????
      if (this.allData.length === 0) {
        this.runData = [];
        return;
      }
      let count = 0;
      const runDataTemp = [[this.allData[0]]];
      for (let i = 1; i < this.allData.length; i += 1) {
        if (this.allData[i][0] !== this.allData[i - 1][0]) {
          count += 1;
          runDataTemp.push([]);
        }
        runDataTemp[count].push(this.allData[i]);
      }
      this.runData = runDataTemp;
    },
  },
};
</script>

<style lang="less" scoped>
.statistics-container {
  margin-bottom: 0.5%;
  background-color: #fff;

  .statistics-title {
    display: flex;
    align-items: center;
    height: auto;
    color: white;
    background-color: white;
  }

  .statistics-content {
    position: relative;
    display: flex;
    flex-direction: row;
    flex-wrap: wrap;
    width: 100%;
    height: 100%;
    margin-left: 1%;

    .allStatisticContainer {
      // width: 24%; // ???????????????
      width: 31%; // ???????????????
      height: 100%;
      margin: 0.8%;
      background-color: white;

      .statisticContaierContent {
        width: 100%;

        .runTag {
          width: 100%;
        }
      }
    }
  }
}

.showClass {
  display: none;
}

.my-label {
  display: flex;
  width: 100%;
  cursor: pointer;

  .triangle {
    position: absolute;
    width: 0;
    height: 0;
    overflow: hidden;
    border-color: transparent transparent transparent #7f7cc1;
    border-style: dashed dashed dashed solid;
    border-top-width: 15px;
    border-right-width: 18px;
    border-bottom-width: 15px;
    border-left-width: 18px;
  }

  .triangle-father {
    position: relative;
  }

  .circle-father {
    position: relative;
    width: 15%;
    height: 30px;
    background-color: #7f7cc1;
  }

  .circle {
    position: absolute;
    top: 50%;
    left: 50%;
    width: 8px;
    height: 8px;
    background-color: white;
    border-radius: 50%;
    transform: translateX(-50%) translateY(-50%);
  }

  .my-text {
    width: 70%;
    height: 30px;
    text-align: center;
    vertical-align: center;
    background-color: #7f7cc1;
  }

  span {
    align-items: center;
    line-height: 30px;
    color: white;
  }
}

.sub .triangle {
  border-color: transparent transparent transparent #b8c6ff;
}

.sub .circle-father {
  background-color: #b8c6ff;
}

.sub .my-text {
  background-color: #b8c6ff;
}

.sub {
  display: flex;
  align-items: center;
  width: 9.5%;
  height: 30px;
  font-family: sans-serif;
  font-size: 12px;
  color: #fff;
  background-size: 100% 100%;
}

.sub1 {
  display: flex;
  align-items: center;
  width: 9.5%;
  height: 30px;
  font-family: sans-serif;
  font-size: 12px;
  color: #fff;
  background-size: 100% 100%;
}

.line1 {
  width: 88%;
  height: 1px;
  background-color: #f4f6ff;
}

.line2 {
  width: 2.5%;
  height: 5px;
  background-color: #625eb3;
}

.linestyle {
  background-color: #b9c6ff;
}
</style>
