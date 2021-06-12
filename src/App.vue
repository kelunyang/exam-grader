<template>
  <v-app>
    <v-sheet>
      <v-dialog
        v-model="syntaxW"
        fullscreen
        hide-overlay
        transition="dialog-bottom-transition"
      >
        <v-card>
          <v-toolbar
            dark
            color="primary"
          >
            <v-btn
              icon
              dark
              @click="importGrading"
            >
              <v-icon>fa-times-circle</v-icon>
            </v-btn>
            <v-toolbar-title>匯出／匯入配分資料</v-toolbar-title>
          </v-toolbar>
          <v-card-text class='d-flex flex-column ma-1'>
            <v-alert type="info" icon="fa-info">這裡的操作方法非常簡單，請你自己把下面這段程式碼全選（Ctrl+A），複製（Ctrl+C）下來，貼到記事本裡存好，下次要用先清除下面文字框的內容後，貼上（Ctrl+V）你備份的設定即可，打完收工</v-alert>
            <tip-tap v-model="exportSyntax" />
          </v-card-text>
        </v-card>
      </v-dialog>
      <v-dialog
        v-model="firsttimeW"
        fullscreen
        hide-overlay
        transition="dialog-bottom-transition"
      >
        <v-card>
          <v-toolbar
            dark
            color="primary"
          >
            <v-btn
              icon
              dark
              @click="firsttimeW = false"
            >
              <v-icon>fa-times-circle</v-icon>
            </v-btn>
            <v-toolbar-title>使用說明</v-toolbar-title>
          </v-toolbar>
          <v-card-text class='d-flex flex-column ma-1'>
            <v-alert type="info" icon="fa-info">如果你完全理解該怎麼匯入，請按左上角X關閉說明</v-alert>
            <p>使用本程式計算Google問卷分數非常簡單，請從Google問卷匯出CSV檔案，丟到本程式中載入，在設定好配分，就可以打完收工了，基本上你就當作是在操作讀卡機軟體吧</p>
            <p>全部最難的步驟可能是從Google問卷匯出CSV，放心，你打開你自己匯出的CSV是亂碼，那是Excel的老bug，本程式可以正常開啟的，你別去加工那個CSV就好了</p>
            <p>本程式邏輯上可以對所有能輸出CSV的問卷作評分，但如果你使用的線上考卷系統沒有CSV匯出功能... 這種不提供原始數據讓人驗算的系統，我也沒辦法啦</p>
            <p>Google問卷匯出CSV的步驟，看下圖</p>
            <img src='@/assets/csvhelp.png' />
          </v-card-text>
        </v-card>
      </v-dialog>
      <v-dialog
        v-model="scoreW"
        fullscreen
        hide-overlay
        transition="dialog-bottom-transition"
      >
        <v-card>
          <v-toolbar
            dark
            color="primary"
          >
            <v-btn
              icon
              dark
              @click="scoreW = false"
            >
              <v-icon>fa-times-circle</v-icon>
            </v-btn>
            <v-toolbar-title>{{ idColumn >= 0 ? workingSheet.name+"["+timeConverter(workingSheet.tick)+"]" : workingSheet.id }}</v-toolbar-title>
            <v-spacer/>
            <v-toolbar-items class='d-flex justify-end'>
              <div class='text-right'>{{ workingSheet.totalScores }}分</div>
            </v-toolbar-items>
          </v-toolbar>
          <v-card-text class='d-flex flex-column ma-1'>
            <v-alert type="info" icon="fa-info">按左上角X關閉檢視</v-alert>
            <v-simple-table>
              <template v-slot:default>
                <thead>
                  <tr>
                    <th class="text-left">
                      考卷答案
                    </th>
                    <th class="text-center">
                      &nbsp;
                    </th>
                    <th
                      class="text-right"
                    >
                      正確答案
                    </th>
                  </tr>
                </thead>
                <tbody>
                  <tr
                    v-for="(item, n) in answerSheet.results"
                    :key="item.id"
                  >
                    <td>{{ workingSheet.results[n].content }}</td>
                    <td :class='workingSheet.results[n].scoreGet === workingSheet.results[n].scoreSet ? "green darken-4" : "red darken-4"' class='white--text'>{{ floatConverter(workingSheet.results[n].scoreGet) }}</td>
                    <td>{{ item.content }}</td>
                  </tr>
                </tbody>
              </template>
            </v-simple-table>
          </v-card-text>
        </v-card>
      </v-dialog>
      <v-dialog
        v-model="gradingW"
        fullscreen
        hide-overlay
        transition="dialog-bottom-transition"
      >
        <v-card>
          <v-toolbar
            dark
            color="primary"
          >
            <v-btn
              icon
              dark
              :disabled='gradingError !== ""'
              @click='closeGrading()'
            >
              <v-icon>fa-times-circle</v-icon>
            </v-btn>
            <v-toolbar-title>設定配分題型</v-toolbar-title>
          </v-toolbar>
          <v-card-text class='d-flex flex-column ma-1 align-self-baseline justify-end'>
            <v-alert type="info" icon="fa-info">設定完成後請按左上角X按鍵關閉對話框</v-alert>
            <v-alert type="error" icon="fa-skull" v-if='gradingError !== ""'>{{ gradingError }}，不修正無法儲存設定！</v-alert>
            <div class='text-body-1'>題型</div>
            <v-select
              outlined
              :items="qTypes"
              v-model="workingGradeSet.type"
              label="請選擇題型"
              hint="目前只支援單選和複選"
            ></v-select>
            <div class='text-body-1'>配分({{ workingGradeSet.score }})</div>
            <v-slider
              v-model="workingGradeSet.score"
              hint="一題幾分呢"
              step="0.5"
              max="10"
              min="1"
            ></v-slider>
            <div class='text-body-1'>選擇題目範圍({{ workingGradeSet.range[0] }}, {{ workingGradeSet.range[1] }})</div>
            <v-range-slider
              v-model="workingGradeSet.range"
              hint="選擇起訖範圍"
              :max="csvHeaders.length - qStart"
              min="1"
            ></v-range-slider>
            <div class='text-body-1'>如果你的複選題只有一題，拖這裡，系統會忽視上面的拉霸({{ workingGradeSet.range[0] }}, {{ workingGradeSet.range[1] }})</div>
            <v-slider
              v-model="singleRange"
              hint="複選題位置"
              :max="csvHeaders.length - qStart"
              min="1"
            ></v-slider>
            <div class='text-body-1' v-if='workingGradeSet.type === 1'>多選題有幾個選項？({{ workingGradeSet.options }})</div>
            <v-slider
              v-model="workingGradeSet.options"
              hint="通常都是5個選項啦"
              max="10"
              min="5"
              v-if='workingGradeSet.type === 1'
            ></v-slider>
          </v-card-text>
        </v-card>
      </v-dialog>
      <v-tabs
        v-model="tab"
        background-color="deep-purple accent-4"
        centered
        dark
        icons-and-text
      >
        <v-tabs-slider></v-tabs-slider>
        <v-tab>
          考卷參數
          <v-icon>fa-cog</v-icon>
        </v-tab>
        <v-tab :disabled='gradingWait'>
          批改結果
          <v-icon>fa-list-ol</v-icon>
        </v-tab>
        <v-tab>
          關於本程式
          <v-icon>fas fa-info-circle</v-icon>
        </v-tab>
      </v-tabs>
      <v-tabs-items v-model="tab">
        <v-tab-item>
          <v-card flat>
            <v-card-text class='d-flex flex-column ma-1'>
              <v-alert type="info" icon="fa-info" v-if='csvData.length === 0'>請選擇你從Google問卷匯出的CSV檔案</v-alert>
              <v-alert type="info" icon="fa-info" v-if='csvData.length > 0'>這張考卷有{{ csvData.length }}題，被你切割為{{ gradingSets.length }}種題型，目前總分為{{ totalScores }}，待批改的答卷有：{{ gradingSheets.length }}張</v-alert>
              <v-alert type="error" icon="fa-skull" v-if='csvError !== ""'>{{ csvError }}</v-alert>
              <v-file-input accept="text/csv" prepend-icon='fa-file-csv' outlined v-model="csvFile" />
              <div class='text-body-1'>題目從哪一欄開始？</div>
              <v-select
                :items="csvHeaders"
                v-model="qStart"
                outlined
                label="題目從哪一欄開始？（本欄位必填）"
                hint="本欄位必填"
              ></v-select>
              <div class='text-body-1'>你的答案卷是第幾份問卷？({{ answerSheetID }})</div>
              <v-slider
                v-model="answerSheetID"
                hint="本題預設值為1"
                :max="csvData.length === 0 ? 2 : csvData.length"
                min="1"
              ></v-slider>
              <div class='text-body-1'>預設分數（如果不設定題型，會以單選題批改，並已預設分數配分）({{ defaultScore }})</div>
              <v-slider
                v-model="defaultScore"
                hint="一題幾分呢"
                step="0.5"
                max="10"
                min="0"
              ></v-slider>
              <div class='text-body-1'>[非必要]學生代號欄位</div>
              <v-select
                outlined
                v-model="idColumn"
                :items="csvHeaders"
                label="學生代號欄位"
                hint="本欄位可留空"
              ></v-select>
              <div class='text-body-1'>[非必要]填寫時間欄位</div>
              <v-select
                v-model="tickColumn"
                :items="csvHeaders"
                outlined
                label="填寫時間欄位"
                hint="本欄位可留空"
              ></v-select>
              <div v-if='tickColumn >= 0' class='text-body-1'>[必要]時間欄位格式（不懂別亂改）</div>
              <v-text-field v-if='tickColumn >= 0' outlined label="時間欄位格式" hint="預設值是按照Google問卷匯出檔去改的，想看請參考moment.js官網，String+Format篇" v-model="dateFormat"></v-text-field>
              <v-btn class='ma-3 red accent-4 white--text' large @click='setGrading(true)'>新增配分和題型</v-btn>
              <v-list-item v-for='grading in gradingSets' :key='grading.id'>
                <v-list-item-content>
                  <v-list-item-title>[{{ grading.type === 0 ? "單選題" : "複選題" }}]第{{ grading.range[0] }}~{{ grading.range[1] }}題，每題{{ grading.score }}分</v-list-item-title>
                </v-list-item-content>
                <v-list-item-action class='flex-shrink-1 d-flex justify-end flex-row'>
                  <v-btn icon outlined @click='setGrading(grading)' class='ma-1'>
                    <v-icon dark color="indigo darken-4">fa-pencil-alt</v-icon>
                  </v-btn>
                  <v-btn icon outlined @click='removeGrading(grading)' class='ma-1'>
                    <v-icon dark color="red accent-4">fa-trash</v-icon>
                  </v-btn>
                </v-list-item-action>
              </v-list-item>
              <v-btn class='ma-3 red accent-4 white--text' large @click='buildGrading'>設定配分</v-btn>
              <v-btn class='ma-3 red accent-4 white--text' large @click='startGrading'>改考卷啦！</v-btn>
              <v-btn class='ma-3 blue accent-4 white--text' large @click='exportGrading'>匯入／匯出配分[非必要動作]</v-btn>
            </v-card-text>
          </v-card>
        </v-tab-item>
        <v-tab-item>
          <v-card flat>
            <v-card-text class='d-flex flex-column ma-1'>
              <v-btn class='ma-3 red accent-4 white--text' large @click='downloadGrading'>點此下載成績單</v-btn>
              <v-simple-table>
                <template v-slot:default>
                  <thead>
                    <tr>
                      <th class="text-left">
                        
                      </th>
                      <th
                        class="text-center"
                      >
                        分數
                      </th>
                      <th
                        class="text-right"
                      >
                        &nbsp;
                      </th>
                    </tr>
                  </thead>
                  <tbody>
                    <tr
                      v-for="item in gradingSheets"
                      :key="item.id"
                    >
                      <td>{{ idColumn >= 0 ? item.name + "[" + timeConverter(item.tick) + "]" : item.id}}</td>
                      <td>{{ item.totalScores }} / {{ totalScores }}</td>
                      <td>
                        <v-btn
                          icon
                          @click='viewResult(item)'
                        >
                          <v-icon>fa-search</v-icon>
                        </v-btn>
                      </td>
                    </tr>
                  </tbody>
                </template>
              </v-simple-table>
            </v-card-text>
          </v-card>
        </v-tab-item>
        <v-tab-item>
          <v-card flat>
            <v-card-text clas='d-flex flex-column ma-1'>
              <div class='text-body-1'>Developer: Kelunyang @ ZLSH</div>
              <div class='text-body-1'>GitHub: <a href="https://github.com/kelunyang/exam-grader" target="_blank">https://github.com/kelunyang/exam-grader</a></div>
              <div class='text-caption'>本程式採用Vue.js + Electorn製作，有Windows編譯版本，但實際上和網頁版效能沒有差異，唯一的差別只有為了保證網頁版也能編譯，上面這個GitHub連結我沒有用electron專用語法，有什麼願望希望實現，歡迎到GitHub丟Issue喔</div>
            </v-card-text>
          </v-card>
        </v-tab-item>
      </v-tabs-items>
    </v-sheet>
  </v-app>
</template>

<style>
@import url('https://fonts.googleapis.com/css?family=Noto+Sans+TC:100,300,400,500,700,900&display=swap');
html {
  scroll-behavior: smooth;
}
#app {
  font-family: 'Noto Sans TC', sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
h1 {
  font-family: 'Noto Sans TC', sans-serif;
  font-weight: 900;
}
</style>

<script>
import _ from 'lodash';
import moment from 'moment';
import { v4 as uuidv4 } from 'uuid';
import stripAllBom from 'strip-all-bom';
import Papa from 'papaparse';
import {Decimal} from 'decimal.js';
import '@fortawesome/fontawesome-free/css/all.css';
import '@fortawesome/fontawesome-free/js/all.js';
import TipTap from './components/TipTap.vue';

export default {
  name: "exam-grader",
  components: {
    TipTap,
  },
  computed: {
    totalScores: function() {
      return _.sumBy(this.answerSheet.results, (item) => {
        return item.scoreSet;
      });
    }
  },
  methods: {
    importGrading: function() {
      let syntax = this.exportSyntax.replace(/<pre><code class="language-javascript">/, '');
      syntax = syntax.replace(/<\/code><\/pre>/, '');
      this.gradingSets = JSON.parse(syntax);
      this.syntaxW = false;
    },
    exportGrading: function() {
      this.exportSyntax = '<pre><code class="language-javascript">' + JSON.stringify(this.gradingSets) + '</code></pre>';
      this.syntaxW = true;
    },
    floatConverter: function(num) {
      let floatNum = new Decimal(num);
      return floatNum.toFixed(1);
    },
    downloadGrading: function() {
      let exportDB = [];
      for(let i=0; i<this.gradingSheets.length; i++) {
        let currentSheet = this.gradingSheets[i];
        let exportItem = {
          '編號': currentSheet.id
        }
        if(this.idColumn >= 0) {
          exportItem['帳號'] = currentSheet.name;
        }
        if(this.tickColumn >= 0) {
          exportItem['日期'] = this.timeConverter(currentSheet.tick);
        }
        exportItem['總分'] = currentSheet.totalScores;
        for(let k=0; k<currentSheet.results.length; k++) {
          let currentAnswer = currentSheet.results[k];
          exportItem[currentAnswer.title] = currentAnswer.type === 0 ? currentAnswer.content : currentAnswer.content.join(',');
        }
        exportDB.push(exportItem);
      }
      var element = document.createElement('a');
      element.setAttribute('href', 'data:text/plain;charset=utf-8,' + "\ufeff"+ Papa.unparse(exportDB));
      element.setAttribute('download', "grade.csv");
      element.style.display = 'none';
      element.click();
    },
    viewResult: function(sheet) {
      this.workingSheet = sheet;
      this.scoreW = true;
    },
    timeConverter: function(time) {
      return moment.unix(time).format("YYYY/MM/DD HH:mm:ss")
    },
    startGrading: function() {
      for(let i=0; i<this.gradingSheets.length; i++) {
        let currentSheet = this.gradingSheets[i];
        for(let k=0; k<this.answerSheet.results.length; k++) {
          let ansAnswer = this.answerSheet.results[k];
          let curAnswer = currentSheet.results[k];
          if(ansAnswer.type === 0) {
            if(ansAnswer.content === curAnswer.content) {
              curAnswer.scoreGet = ansAnswer.scoreSet;
            }
          } else {
            let diffCAns = _.differenceWith(curAnswer.content, ansAnswer.content, (curItem, ansItem) => {
              return curItem === ansItem;
            });
            let diffAAns = _.differenceWith(ansAnswer.content, curAnswer.content, (curItem, ansItem) => {
              return curItem === ansItem;
            });
            let wrongAns = diffCAns.length + diffAAns.length;
            if(wrongAns < 3) {
              curAnswer.scoreGet = ((ansAnswer.content.length - (2 * wrongAns)) / ansAnswer.content.length) * ansAnswer.scoreSet;
            } else {
              curAnswer.scoreGet = 0;
            }
          }
        }
        let totalScores = new Decimal(_.sumBy(currentSheet.results, (item) => {
          return item.scoreGet;
        }));
        currentSheet.totalScores = totalScores.toFixed(1);
      }
      this.gradingWait = false;
      this.tab = 1;
    },
    buildGrading: function() {
      this.sheetData = [];
      for(let k=0; k<this.csvData.length; k++) {
        let currentCSV = this.csvData[k];
        let currentSheet = {
          id: uuidv4(),
          name: this.idColumn >= 0 ? currentCSV[this.csvHeaders[this.idColumn].text] : "",
          tick: this.tickColumn >= 0 ? moment(currentCSV[this.csvHeaders[this.tickColumn].text], this.dateFormat).unix() : 0,
          totalScores: 0,
          results: []
        };
        for(let i=this.qStart; i<this.csvHeaders.length; i++) {
          let gradeSetted = false;
          let currentAnswer = {
            id: uuidv4(),
            title: this.csvHeaders[i].text,
            scoreGet: 0
          }
          for(let r=0; r<this.gradingSets.length; r++) {
            let currentSet = this.gradingSets[r];
            let currentRange = [];
            for(let o=currentSet.range[0]; o<=currentSet.range[1]; o++) {
              currentRange.push(o);
            }
            let intersection = _.intersectionWith([i - this.qStart + 1], currentRange, (aItem, bItem) => {
              return aItem === bItem;
            })
            if(intersection.length > 0) {
              gradeSetted = true;
              currentAnswer.type = currentSet.type;
              currentAnswer.scoreSet = currentSet.score;
              if(currentSet.type === 1) {
                currentAnswer.content = currentCSV[currentAnswer.title].split(';');
                if(currentAnswer.content.length < currentSet.options) {
                  let subtract = currentSet.options - currentAnswer.content.length;
                  for(let n=0; n<subtract; n++) {
                    currentAnswer.content.push("");
                  }
                }
              } else {
                currentAnswer.content = currentCSV[currentAnswer.title];
              }
            }
          }
          if(!gradeSetted) {
            currentAnswer.type = 0;
            currentAnswer.options = 0;
            currentAnswer.scoreSet = this.defaultScore;
            currentAnswer.content = currentCSV[currentAnswer.title];
          }
          currentSheet.results.push(currentAnswer);
        }
        this.sheetData.push(currentSheet);
      }
      this.answerSheet = this.sheetData[this.answerSheetID - 1];
      this.gradingSheets = _.filter(this.sheetData, (sheet) => {
        return sheet.id !== this.answerSheet.id
      });
    },
    removeGrading: function(gradingSet) {
      this.gradingSets = _.filter(this.gradingSets, (item) => {
        return item.id !== gradingSet.id;
      })
    },
    closeGrading: function() {
      this.gradingW = false;
    },
    setGrading: function(grade) {
      if(grade === true) {
        let newItem = {
          id: uuidv4(),
          score: 0,
          range: [1, this.csvHeaders.length - this.qStart],
          options: 5,
          type: 0
        };
        this.gradingSets.push(newItem);
        this.singleRange = -1;
        this.workingGradeSet = newItem;
      } else {
        this.workingGradeSet = grade;
        if(grade.range[0] === grade.range[1]) {
          this.singleRange = grade.range[0];
        }
      }
      this.gradingW = true;
    }
  },
  watch: {
    singleRange: function() {
      if(this.singleRange > 0) {
        this.workingGradeSet.range = [this.singleRange, this.singleRange];
      }
    },
    'workingGradeSet.range': {
      immediate: true,
      handler() {
        let ranges = [];
        let errorCount = 0;
        for(let i=0; i<this.gradingSets.length; i++) {
          let currentSet = this.gradingSets[i];
          if(currentSet.id !== this.workingGradeSet.id) {
            let range = [];
            for(let k=currentSet.range[0]; k<=currentSet.range[1]; k++) {
              range.push(k);
            }
            ranges.push(range);
          }
        }
        let workingRange = [];
        for(let i=this.workingGradeSet.range[0]; i<=this.workingGradeSet.range[1]; i++) {
          workingRange.push(i);
        }
        for(let i=0; i<ranges.length; i++) {
          let currentRange = ranges[i];
          let intersection = _.intersectionWith(workingRange, currentRange, (aItem, bItem) => {
            return aItem === bItem;
          });
          if(intersection.length > 0) {
            this.gradingError = "您選的題目區段和既有重疊！";
            errorCount++;
          }
        }
        if(errorCount === 0) {
          this.gradingError = "";
        }
      }
    },
    csvFile: {
      immediate: true,
      async handler () {
        let oriobj = this;
        if (this.csvFile !== undefined) {
          let reader = new FileReader();
          reader.readAsText(oriobj.csvFile);
          reader.onload = ((content) => { 
            try {
              content = stripAllBom(content.target.result);
              Papa.parse(content, {
                header: true,
                skipEmptyLines: true,
                complete: async function(result) {
                  if(result.data.length > 0) {
                    let headers = Object.keys(result.data[0]);
                    oriobj.csvHeaders = [];
                    for(let i=0; i<headers.length; i++) {
                      oriobj.csvHeaders.push({
                        text: headers[i],
                        value: i
                      });
                    }
                    oriobj.csvData = result.data;
                  } else {
                    oriobj.csvError = 'CSV是空的';
                  }
                }
              });
            } catch(e) {
              oriobj.csvError = '匯入發生錯誤（' + JSON.stringify(e) +'）';
            }
          });
        }
      }
    }
  },
  data: () => ({
    exportSyntax: '<pre><code class="language-javascript">{ "json": true }</code></pre>',
    syntaxW: false,
    syntaxEditor: null,
    dateFormat: "YYYY/MM/DD h:mm:ss a Z",
    gradingWait: true,
    singleRange: -1,
    tab: 0,
    gradingSheets: [],
    answerSheet: {
      results: []
    },
    defaultScore: 2,
    sheetData: [],
    gradingError: '',
    workingSheet: {
      id: '',
      name: '',
      tick: 0,
      score: 0,
      results: []
    },
    qTypes: [
      { text:"單選題", value: 0},
      { text:"複選題", value: 1}
    ],
    gradingSets: [],
    gradingW: false,
    scoreW: false,
    firsttimeW: true,
    workingGradeSet: {
      id: uuidv4(),
      score: 0,
      range: [1, 1],
      options: 5,
      type: 0
    },
    idColumn: -1,
    tickColumn: -1,
    answerSheetID: 1,
    qStart: 0,
    csvHeaders: [],
    csvData: [],
    csvError: "",
    csvFile: undefined
  }),
};
</script>
