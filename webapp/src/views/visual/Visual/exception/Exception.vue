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
  <div>
    <div class="temp">
      <div id="excepDisplay" :class="['display-panel']">
        <div
          v-for="(item, index) in allData"
          v-show="excepRunShow[item[0]]"
          :key="index"
          class="excepContDiv"
        >
          <excepContainer :oneData="item" :index="index" :oneAllStep="allStep[index]" />
        </div>
      </div>
    </div>
  </div>
</template>
<script>
import { createNamespacedHelpers } from 'vuex';
import excepContainer from './excepContainer';

const {
  mapActions: mapExceptionActions,
  mapGetters: mapExceptionGetters,
  mapMutations: mapExceptionMutations,
} = createNamespacedHelpers('Visual/exception');
const { mapState: mapLayoutStates } = createNamespacedHelpers('Visual/layout');
export default {
  components: {
    excepContainer,
  },
  data() {
    return {
      allData: [],
      allStep: [],
      excepRunShow: {},
    };
  },
  computed: {
    ...mapExceptionGetters([
      'getRun',
      'getAllStep',
      'getAllData',
      'getInitStateFlag',
      'getErrorMessage',
      'getFreshFlag',
    ]),
    ...mapLayoutStates(['userSelectRunFile']),
  },
  watch: {
    getAllData(val) {
      this.allData = val;
    },
    getAllStep(val) {
      this.allStep = val;
      this.fetchAllData();
    },
    userSelectRunFile() {
      this.setRunShow();
    },
    getErrorMessage(val) {
      this.$message({
        message: val,
        type: 'error',
      });
    },
  },
  created() {
    // ???????????????????????????????????????????????????????????????exception.js??????????????????????????????
    // ???????????????????????????????????????????????????????????????????????????????????????
    if (this.getRun.length === 0) return; // ??????????????????????????????
    this.setRunShow();
    if (this.getAllStep.length === 0) {
      this.fetchAllStep();
    } else if (this.getAllStep.length !== 0 && this.getAllData.length === 0) {
      this.fetchAllData();
    } else if (this.getAllStep.length !== 0 && this.getAllData.length !== 0) {
      this.allStep = this.getAllStep;
      this.allData = this.getAllData;
    }
  },
  methods: {
    ...mapExceptionActions(['fetchAllStep', 'fetchAllData']),
    ...mapExceptionMutations([
      'setInitStateFlag',
      'setFreshFlag',
      'setRectCurInfo',
      'setCurIqrTimes',
    ]),
    setRunShow() {
      const stateTemp = [];
      for (let i = 0; i < this.getRun.length; i += 1) {
        stateTemp[this.getRun[i]] = false;
      }
      for (let i = 0; i < this.userSelectRunFile.length; i += 1) {
        stateTemp[this.userSelectRunFile[i]] = true;
      }
      this.excepRunShow = stateTemp;
      if (this.userSelectRunFile.length === 0) {
        // ??????????????????run???????????????????????????
        this.setRectCurInfo(['', '', '', '', '', '', '']);
        this.setCurIqrTimes(['', '', '', '', '', '', '']);
      }
    },
  },
};
</script>

<style lang="less" scoped>
.temp {
  width: 100%;
  height: 100%;
  overflow-y: hidden;
  background-color: white;
}

.display-panel {
  height: 97.5%;
  margin: 1% 1% 0 1%;
  overflow-y: auto;
  background-color: white;
  border-radius: 5px 5px 0 0;
  box-shadow: rgba(0, 0, 0, 0.3) 0 0 10px;
}
</style>
