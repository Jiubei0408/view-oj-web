<template>
  <div>
    <el-dialog title="Edit Monitor" :visible.sync="addBoxShow" :append-to-body="true" width="600px">
      <div>
        <el-input placeholder="The Title of The New Problem Set" v-model="newSetTitle"
                  prefix-icon="el-icon-edit-outline"></el-input>
      </div>
      <el-divider><i class="el-icon-document"/></el-divider>
      <el-tabs v-model="tabActive" type="card">
        <el-tab-pane label="比赛" name="contest">
          <div style="padding: 20px">
            <el-input v-model="contest.contestId" placeholder="ContestID">
              <el-select v-model="contest.oj" style="width: 150px" slot="prepend">
                <el-option v-for="oj in contest_oj_list" :key="oj.id" :label="oj.name" :value="oj.id"/>
              </el-select>
            </el-input>
          </div>
        </el-tab-pane>
        <el-tab-pane label="题集" name="problem-set">
          <div class="itemBox">
            <template v-for="(item,index) in problemSet">
              <div :key="index" class="itemItem">
                <b style="margin-right: 10px">{{ String.fromCharCode((65 + index)) }}</b>
                <el-input v-model="item.pid" placeholder="Pro. Num">
                  <el-select v-model="item.oj" placeholder="OJ" style="width: 150px" slot="prepend">
                    <el-option v-for="OJ in ojList" :key="OJ.id" :label="OJ.name" :value="OJ.name"></el-option>
                  </el-select>
                  <el-select v-model="item.difficulty" style="width: 120px" slot="append">
                    <el-option v-for="i in Array(6).keys()" :key="i" :value="i" :label="getDifficultyDescribe(i)"/>
                  </el-select>
                </el-input>
                <el-button type="danger" icon="el-icon-delete" size="mini" circle style="margin-left: 10px;"
                           @click="delItem(item)"></el-button>
              </div>
            </template>
            <div class="itemItem">
              <el-button icon="el-icon-plus" round @click="addProblem">Problem</el-button>
            </div>
          </div>
        </el-tab-pane>
      </el-tabs>
      <el-divider><i class="el-icon-user-solid"/></el-divider>
      <el-tree
          :data="user_data"
          show-checkbox
          check-on-click-node
          :expand-on-click-node="false"
          ref="user_tree"/>
      <div slot="footer" class="dialog-footer">
        <el-button @click="initProblemSet">Cancel</el-button>
        <el-button type="primary" @click="submitCreate">Submit</el-button>
      </div>
    </el-dialog>
    <el-card shadow="hover" class="tableBox">
      <div class="tableTitle">
        <b>The Monitor of The Problem Set</b>
      </div>
      <div class="centerBox" v-if="this.$store.state.user.permission === 1">
        <el-button-group>
          <el-tooltip effect="dark" content="Add New Problem Set" placement="top">
            <el-button icon="el-icon-plus" size="medium" @click="addBoxShow = true" round></el-button>
          </el-tooltip>
          <template v-if="isAdminMode">
            <el-tooltip effect="dark" content="Close Admin Mode" placement="top">
              <el-button icon="el-icon-unlock" size="medium" @click="isAdminMode = false" round
                         type="danger"></el-button>
            </el-tooltip>
          </template>
          <template v-else>
            <el-tooltip effect="dark" content="Open Admin Mode" placement="top">
              <el-button icon="el-icon-lock" size="medium" @click="isAdminMode = true" round></el-button>
            </el-tooltip>
          </template>
          <el-tooltip effect="dark" content="Refresh All Data" placement="top">
            <el-button icon="el-icon-refresh" size="medium" @click="$store.commit('updateAllData')"
                       round></el-button>
          </el-tooltip>
        </el-button-group>
      </div>
      <el-collapse accordion @change="openItem">
        <template v-for="item in SetList">
          <el-collapse-item :name="item.id" :key="item.id">
            <template slot="title">
              <i class="el-icon-collection-tag" style="margin-left: 10px;color: #409EFF"></i>
              <el-divider direction="vertical"></el-divider>
              <template v-if="isAdminMode">
                <el-popover placement="top" title="Rename" trigger="hover" v-model="item.renameShow"
                            width="400">
                  <i class="el-icon-edit" slot="reference" style="color: #E6A23C"></i>
                  <el-input v-model="item.name" prefix-icon="el-icon-edit"></el-input>
                  <div style="text-align: right; margin-top: 10px">
                    <el-button size="mini" type="text" @click="item.renameShow = false"
                               style="color: black">Cancel
                    </el-button>
                    <el-button type="warning" size="mini"
                               @click="renameSet(item); item.renameShow = false">Submit
                    </el-button>
                  </div>
                </el-popover>
                <el-divider direction="vertical"></el-divider>
                <el-popover placement="top" trigger="hover" v-model="item.delShow">
                  <i class="el-icon-delete" slot="reference" style="color: #F56C6C"></i>
                  <p style="text-align: center">Are you sure you want to delete <br/><br/><b>{{ item.name }}</b>
                    ?<br/></p>
                  <div style="text-align: right; margin-top: 10px">
                    <el-button size="mini" type="text" @click="item.delShow = false"
                               style="color: black">Cancel
                    </el-button>
                    <el-button type="danger" size="mini"
                               @click="delSet(item.id); item.delShow = false">Delete
                    </el-button>
                  </div>
                </el-popover>
                <el-divider direction="vertical"></el-divider>
              </template>
              {{ item.name }}
            </template>
            <div>
              <template v-if="activeNames === item.id">
                <template v-if="isLoading">
                  <i class="el-icon-loading"></i> Loading...
                </template>
                <template v-else>
                  <monitor-table :proData="proData" :proList="proList"></monitor-table>
                </template>
              </template>
            </div>
          </el-collapse-item>
        </template>
      </el-collapse>
    </el-card>
  </div>
</template>

<script>
import MonitorTable from '@/components/pageitem/monitor-table'

export default {
  name: 'monitor-page',
  data() {
    return {
      api: this.$store.state.api,
      SetList: [],
      activeNames: -1,
      isLoading: true,
      proData: [],
      proList: [],
      ojList: [],
      contest_oj_list: [],
      addBoxShow: false,
      newSetTitle: '',
      problemSet: [],
      contest: {
        oj: '',
        contestId: ''
      },
      isAdminMode: false,
      user_loading: false,
      user_data: [],
      tabActive: 'contest'
    }
  },
  components: {
    'monitor-table': MonitorTable
  },
  methods: {
    getList() {
      this.SetList = []
      var that = this
      that.$http.get(this.api + '/v2/problem_set/summary')
          .then(data => {
            this.SetList = data.data.data
          })
          .catch(function (error) {
            if (error.response) {
              that.$message.error(error.response.data.msg)
            }
          })
    },
    getTableData(activeNames) {
      var that = this
      that.$http.get(this.api + '/v2/problem_set/' + activeNames)
          .then(data => {
            this.proData = data.data.data.detail
            this.proList = data.data.data.problem_list
            this.isLoading = false
          })
          .catch(function (error) {
            if (error.response) {
              that.$message.error(error.response.data.msg)
            }
          })
    },
    openItem(activeNames) {
      this.activeNames = activeNames
      if (!activeNames) {
        this.proData = []
        return
      }
      this.isLoading = true
      this.getTableData(activeNames)
    },
    getOJList() {
      var that = this
      that.$http.get(this.api + '/v2/problem_set/valid_oj')
          .then(data => {
            this.ojList = []
            for (var item of data.data.data) {
              if (item.name === 'vjudge') continue
              this.ojList.push({
                id: item.id,
                name: item.name
              })
            }
          })
          .catch(function (error) {
            if (error.response) {
              that.$message.error(error.response.data.msg)
            }
          })
      that.$http.get(this.api + '/v2/problem_set/valid_contest_oj')
          .then(data => {
            this.contest_oj_list = []
            for (let item of data.data.data) {
              this.contest_oj_list.push({
                id: item.id,
                name: item.name
              })
            }
          })
          .catch(error => {
            if (error.response) that.$message.error(error.response.data.msg)
          })
    },
    initProblemSet() {
      this.newSetTitle = ''
      this.problemSet = []
      this.addBoxShow = false
    },
    addProblem() {
      var num = this.problemSet.length
      if (num === 0) {
        this.problemSet.push({
          oj: null,
          pid: '',
          difficulty: 0
        })
      } else {
        let oldpid = this.problemSet[num - 1].pid
        let newpid
        if (oldpid === '') newpid = ''
        else if (isNaN(oldpid)) newpid = oldpid.substr(0, oldpid.length - 1) + String.fromCharCode(oldpid.charCodeAt(oldpid.length - 1) + 1)
        else newpid = (parseInt(oldpid) + 1).toString()
        this.problemSet.push({
          oj: this.problemSet[num - 1].oj,
          pid: newpid,
          difficulty: 0
        })
      }
    },
    delItem(item) {
      var index = this.problemSet.indexOf(item)
      if (index !== -1) {
        this.problemSet.splice(index, 1)
      }
    },
    submitCreate() {
      let checked_nodes = this.$refs.user_tree.getCheckedNodes()
      let checked_users = []
      for (let i of checked_nodes) {
        if (i.children === undefined) checked_users.push(i.username)
      }
      const that = this
      if (!that.newSetTitle) {
        that.$message.error('Title cannot be empty!')
        return
      }
      if (checked_users.length === 0) {
        that.$message.error('Please select users')
        return
      }
      if (this.tabActive === 'contest') {
        that.$http.post(this.api + '/v2/problem_set/add_contest', {
          name: that.newSetTitle,
          contest_oj_id: that.contest.oj,
          contest_id: that.contest.contestId,
          user_list: JSON.stringify(checked_users)
        })
            .then(data => {
              that.$message.success(data.data.msg)
              that.initProblemSet()
              that.getList()
            })
            .catch(function (error) {
              if (error.response) {
                that.$message.error(error.response.data.msg)
              }
            })
      }
      if (this.tabActive === 'problem-set') {
        for (let item of that.problemSet) {
          if (!(item.oj && item.pid)) {
            that.$message.error('Problem cannot be empty!')
            return
          }
        }
        if (that.problemSet.length === 0) {
          that.$message.error('ProblemSet cannot be empty!')
          return
        }
        let problems = []
        for (let prob of that.problemSet) {
          problems.push({
            problem: prob.oj + '-' + prob.pid,
            difficulty: prob.difficulty
          })
        }
        that.$http.post(this.api + '/v2/problem_set/add_problem_set', {
          name: that.newSetTitle,
          problem_list: JSON.stringify(Array.from(problems)),
          user_list: JSON.stringify(checked_users)
        })
            .then(data => {
              that.$message.success(data.data.msg)
              that.initProblemSet()
              that.getList()
            })
            .catch(function (error) {
              if (error.response) {
                that.$message.error(error.response.data.msg)
              }
            })
      }
    },
    delSet(setId) {
      const that = this
      that.$http.delete(this.api + '/v2/problem_set/' + setId)
          .then(data => {
            that.$message.success(data.data.msg)
            that.getList()
          })
          .catch(function (error) {
            if (error.response) {
              that.$message.error(error.response.data.msg)
            }
          })
    },
    renameSet(SET) {
      const that = this
      that.$http.put(this.api + '/v2/problem_set/' + SET.id, {
        name: SET.name
      })
          .then(data => {
            that.$message.success(data.data.msg)
            that.getList()
          })
          .catch(function (error) {
            if (error.response) {
              that.$message.error(error.response.data.msg)
            }
          })
    },
    getDifficultyDescribe(difficulty) {
      switch (difficulty) {
        case 0:
          return '未评级'
        case 1:
          return '普通'
        case 2:
          return '稀有'
        case 3:
          return '史诗'
        case 4:
          return '传说'
        case 5:
          return '神话'
        default:
          return 'ERROR'
      }
    },
    getUserData() {
      this.user_loading = true
      if (!this.$store.state.Userlist) this.$store.commit('updateUserlist')
      this.initUserData()
    },
    initUserData() {
      if (this.$store.state.Userlist) {
        this.user_loading = false
        let usermap = new Map()
        for (let item of this.$store.state.Userlist) {
          let user = item[1]
          if (user.status === 0) continue
          if (!usermap.has(user.group)) usermap.set(user.group, [])
          user.label = user.nickname
          usermap.get(user.group).push(user)
        }
        this.user_data = []
        for (let item of usermap) {
          this.user_data.push({
            label: item[0],
            children: item[1]
          })
        }
        this.user_data = [{
          label: '全部',
          children: this.user_data
        }]
      } else {
        setTimeout(() => {
          this.initUserData()
        }, 500)
      }
    }
  },
  created() {
    document.title = "Monitor - viewOJ"
    this.getList()
    this.getOJList()
    this.getUserData()
  }
}
</script>

<style scoped>
.tableTitle {
  text-align: center;
  font-size: 25px;
  margin-top: 30px;
  margin-bottom: 30px;
}

.tableBox {
  position: relative;
  left: 50%;
  transform: translate(-50%, 0);
}

.centerBox {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-top: 30px;
  margin-bottom: 30px;
}

.itemBox {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 20px;
  flex-direction: column;
}

.itemItem {
  width: 90%;
  margin-top: 20px;
  display: flex;
  align-items: center;
  justify-content: space-around
}

/deep/ .el-tree-node__children .el-tree-node__children .el-tree-node__content {
  float: left;
  width: 20%;
  padding: 0 !important;
  margin-left: 20px;
}

/deep/ .el-input-group__append, /deep/ .el-input-group__prepend {
  background-color: transparent;
}

</style>
