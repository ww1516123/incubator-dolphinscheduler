<template>
  <div class="form-model-model" v-clickoutside="_handleClose">
    <div class="title-box">
      <span class="name">{{$t('Current node settings')}}</span>
      <span class="go-subtask">
        <!-- Component can't pop up box to do component processing -->
        <m-log :item="backfillItem">
          <template slot="history"><a href="javascript:" @click="_seeHistory" ><i class="iconfont">&#xe6ee;</i><em>{{$t('View history')}}</em></a></template>
          <template slot="log"><a href="javascript:"><i class="iconfont">&#xe691;</i><em>{{$t('View log')}}</em></a></template>
        </m-log>
        <a href="javascript:" @click="_goSubProcess" v-if="_isGoSubProcess"><i class="iconfont">&#xe600;</i><em>{{$t('Enter this child node')}}</em></a>
      </span>
    </div>
    <div class="content-box" v-if="isContentBox">
      <div class="from-model">
        <!-- Node name -->
        <div class="clearfix list">
          <div class="text-box"><span>{{$t('Node name')}}</span></div>
          <div class="cont-box">
            <label class="label-box">
              <x-input
                type="text"
                v-model="name"
                :disabled="isDetails"
                :placeholder="$t('Please enter name(required)')"
                maxlength="100"
                @on-blur="_verifName()"
                autocomplete="off">
              </x-input>
            </label>
          </div>
        </div>

        <!-- Running sign -->
        <div class="clearfix list">
          <div class="text-box"><span>{{$t('Run flag')}}</span></div>
          <div class="cont-box">
            <label class="label-box">
              <x-radio-group v-model="runFlag" >
                <x-radio :label="'NORMAL'" :disabled="isDetails">{{$t('Normal')}}</x-radio>
                <x-radio :label="'FORBIDDEN'" :disabled="isDetails">{{$t('Prohibition execution')}}</x-radio>
              </x-radio-group>
            </label>
          </div>
        </div>

        <!-- desc -->
        <div class="clearfix list">
          <div class="text-box">
            <span>{{$t('Description')}}</span>
          </div>
          <div class="cont-box">
            <label class="label-box">
              <x-input
                resize
                :autosize="{minRows:2}"
                type="textarea"
                :disabled="isDetails"
                v-model="desc"
                :placeholder="$t('Please enter description')"
                autocomplete="off">
              </x-input>
            </label>
          </div>
        </div>

        <!-- Task priority -->
        <div class="clearfix list">
          <div class="text-box">
            <span>{{$t('Task priority')}}</span>
          </div>
          <div class="cont-box">
            <span class="label-box" style="width: 193px;display: inline-block;">
              <m-priority v-model="taskInstancePriority"></m-priority>
            </span>
            <span class="text-b">{{$t('Worker group')}}</span>
            <m-worker-groups v-model="workerGroupId"></m-worker-groups>
          </div>
        </div>

        <!-- Number of failed retries -->
        <div class="clearfix list" v-if="taskType !== 'SUB_PROCESS'">
          <div class="text-box">
            <span>{{$t('Number of failed retries')}}</span>
          </div>
          <div class="cont-box">
            <m-select-input v-model="maxRetryTimes" :list="[0,1,2,3,4]">
            </m-select-input>
            <span>({{$t('Times')}})</span>
            <span class="text-b">{{$t('Failed retry interval')}}</span>
            <m-select-input v-model="retryInterval" :list="[1,10,30,60,120]">
            </m-select-input>
            <span>({{$t('Minute')}})</span>
          </div>
        </div>

        <!-- Task timeout alarm -->
        <m-timeout-alarm
          ref="timeout"
          :backfill-item="backfillItem"
          @on-timeout="_onTimeout">
        </m-timeout-alarm>

        <!-- shell node -->
        <m-shell
          v-if="taskType === 'SHELL'"
          @on-params="_onParams"
          ref="SHELL"
          :backfill-item="backfillItem">
        </m-shell>
        <!-- sub_process node -->
        <m-sub-process
          v-if="taskType === 'SUB_PROCESS'"
          @on-params="_onParams"
          @on-set-process-name="_onSetProcessName"
          ref="SUB_PROCESS"
          :backfill-item="backfillItem">
        </m-sub-process>
        <!-- procedure node -->
        <m-procedure
          v-if="taskType === 'PROCEDURE'"
          @on-params="_onParams"
          ref="PROCEDURE"
          :backfill-item="backfillItem">
        </m-procedure>
        <!-- sql node -->
        <m-sql
          v-if="taskType === 'SQL'"
          @on-params="_onParams"
          ref="SQL"
          :create-node-id="id"
          :backfill-item="backfillItem">
        </m-sql>
        <!-- spark node -->
        <m-spark
          v-if="taskType === 'SPARK'"
          @on-params="_onParams"
          ref="SPARK"
          :backfill-item="backfillItem">
        </m-spark>
        <m-flink
          v-if="taskType === 'FLINK'"
          @on-params="_onParams"
          ref="FLINK"
          :backfill-item="backfillItem">
        </m-flink>
        <!-- mr node -->
        <m-mr
          v-if="taskType === 'MR'"
          @on-params="_onParams"
          ref="MR"
          :backfill-item="backfillItem">
        </m-mr>
        <!-- python node -->
        <m-python
          v-if="taskType === 'PYTHON'"
          @on-params="_onParams"
          ref="PYTHON"
          :backfill-item="backfillItem">
        </m-python>
        <!-- dependent node -->
        <m-dependent
          v-if="taskType === 'DEPENDENT'"
          @on-dependent="_onDependent"
          ref="DEPENDENT"
          :backfill-item="backfillItem">
        </m-dependent>

      </div>
    </div>
    <div class="bottom-box">
      <div class="submit" style="background: #fff;">
        <x-button type="text" @click="close()"> {{$t('Cancel')}} </x-button>
        <x-button type="primary" shape="circle" :loading="spinnerLoading" @click="ok()" :disabled="isDetails">{{spinnerLoading ? 'Loading...' : $t('Confirm add')}} </x-button>
      </div>
    </div>
  </div>
</template>
<script>
  import _ from 'lodash'
  import mLog from './log'
  import mMr from './tasks/mr'
  import mSql from './tasks/sql'
  import i18n from '@/module/i18n'
  import mShell from './tasks/shell'
  import mSpark from './tasks/spark'
  import mFlink from './tasks/flink'
  import mPython from './tasks/python'
  import JSP from './../plugIn/jsPlumbHandle'
  import mProcedure from './tasks/procedure'
  import mDependent from './tasks/dependent'
  import mSubProcess from './tasks/sub_process'
  import mSelectInput from './_source/selectInput'
  import mTimeoutAlarm from './_source/timeoutAlarm'
  import mWorkerGroups from './_source/workerGroups'
  import clickoutside from '@/module/util/clickoutside'
  import disabledState from '@/module/mixin/disabledState'
  import { isNameExDag, rtBantpl } from './../plugIn/util'
  import mPriority from '@/module/components/priority/priority'

  export default {
    name: 'form-model',
    data () {
      return {
        // loading
        spinnerLoading: false,
        // node name
        name: ``,
        // desc
        desc: '',
        // Node echo data
        backfillItem: {},
        // Resource(list)
        resourcesList: [],
        // dependence
        dependence: {},
        // Current node params data
        params: {},
        // Running sign
        runFlag: 'NORMAL',
        // The second echo problem caused by the node data is specifically which node hook caused the unfinished special treatment
        isContentBox: false,
        // Number of failed retries
        maxRetryTimes: '0',
        // Failure retry interval
        retryInterval: '1',
        // Task timeout alarm
        timeout: {},
        // Task priority
        taskInstancePriority: 'MEDIUM',
        // worker group id
        workerGroupId: -1
      }
    },
    /**
     * Click on events that are not generated internally by the component
     */
    directives: { clickoutside },
    mixins: [disabledState],
    props: {
      id: Number,
      taskType: String,
      self: Object
    },
    methods: {
      /**
       * depend
       */
      _onDependent (o) {
        this.dependence = Object.assign(this.dependence, {}, o)
      },
      /**
       * Task timeout alarm
       */
      _onTimeout (o) {
        this.timeout = Object.assign(this.timeout, {}, o)
      },
      /**
       * Click external to close the current component
       */
      _handleClose () {
        this.close()
      },
      /**
       * Jump to task instance
       */
      _seeHistory () {
        this.self.$router.push({
          name: 'task-instance',
          query: {
            processInstanceId: this.self.$route.params.id,
            taskName: this.backfillItem.name
          }
        })
        this.$modal.destroy()
      },
      /**
       * Enter the child node to judge the process instance or the process definition
       * @param  type = instance
       */
      _goSubProcess () {
        if (_.isEmpty(this.backfillItem)) {
          this.$message.warning(`${i18n.$t('The newly created sub-Process has not yet been executed and cannot enter the sub-Process')}`)
          return
        }
        if (this.router.history.current.name === 'projects-instance-details') {
          let stateId = $(`#${this.id}`).attr('data-state-id') || null
          if (!stateId) {
            this.$message.warning(`${i18n.$t('The task has not been executed and cannot enter the sub-Process')}`)
            return
          }
          this.store.dispatch('dag/getSubProcessId', { taskId: stateId }).then(res => {
            this.$emit('onSubProcess', {
            subProcessId: res.data.subProcessInstanceId,
            fromThis: this
          })
        }).catch(e => {
            this.$message.error(e.msg || '')
        })
        } else {
          this.$emit('onSubProcess', {
            subProcessId: this.backfillItem.params.processDefinitionId,
            fromThis: this
          })
        }
      },
      /**
       * return params
       */
      _onParams (o) {
        this.params = Object.assign(this.params, {}, o)
      },
      /**
       * verification name
       */
      _verifName () {
        if (!_.trim(this.name)) {
          this.$message.warning(`${i18n.$t('Please enter name (required)')}`)
          return false
        }
        if (this.name === this.backfillItem.name) {
          return true
        }
        // Name repeat depends on dom backfill dependent store
        if (isNameExDag(this.name, _.isEmpty(this.backfillItem) ? 'dom' : 'backfill')) {
          this.$message.warning(`${i18n.$t('Name already exists')}`)
          return false
        }
        return true
      },
      /**
       * Global verification procedure
       */
      _verification () {
        // Verify name
        if (!this._verifName()) {
          return
        }
        // Verify task alarm parameters
        if (!this.$refs['timeout']._verification()) {
          return
        }
        // Verify node parameters
        if (!this.$refs[this.taskType]._verification()) {
          return
        }

        $(`#${this.id}`).find('span').text(this.name)

        // Store the corresponding node data structure
        this.$emit('addTaskInfo', {
          item: {
            type: this.taskType,
            id: this.id,
            name: this.name,
            params: this.params,
            desc: this.desc,
            runFlag: this.runFlag,
            dependence: this.dependence,
            maxRetryTimes: this.maxRetryTimes,
            retryInterval: this.retryInterval,
            timeout: this.timeout,
            taskInstancePriority: this.taskInstancePriority,
            workerGroupId: this.workerGroupId
          },
          fromThis: this
        })

        // set run flag
        this._setRunFlag()
      },
      /**
       * Sub-workflow selected node echo name
       */
      _onSetProcessName (name) {
        this.name = name
      },
      /**
       *  set run flag
       */
      _setRunFlag () {
        let dom = $(`#${this.id}`).find('.ban-p')
        dom.html('')
        if (this.runFlag === 'FORBIDDEN') {
          dom.append(rtBantpl())
        }
      },
      /**
       * Submit verification
       */
      ok () {
        this._verification()
      },
      /**
       * Close and destroy component and component internal events
       */
      close () {
        let flag = false
        // Delete node without storage
        if (!this.backfillItem.name) {
          flag = true
        }
        this.isContentBox = false
        // flag Whether to delete a node this.$destroy()
        this.$emit('close', {
          flag: flag,
          fromThis: this
        })
      }
    },
    watch: {

    },
    created () {
      // Unbind copy and paste events
      JSP.removePaste()
      // Backfill data
      let taskList = this.store.state.dag.tasks
      let o = {}
      if (taskList.length) {
        taskList.forEach(v => {
          if (v.id === this.id) {
          o = v
          this.backfillItem = v
        }
      })
        // Non-null objects represent backfill
        if (!_.isEmpty(o)) {
          this.name = o.name
          this.taskInstancePriority = o.taskInstancePriority
          this.runFlag = o.runFlag || 'NORMAL'
          this.desc = o.desc
          this.maxRetryTimes = o.maxRetryTimes
          this.retryInterval = o.retryInterval
          this.workerGroupId = o.workerGroupId
        }
      }
      this.isContentBox = true
    },
    mounted () {

    },
    updated () {
    },
    beforeDestroy () {
    },
    destroyed () {
    },
    computed: {
      /**
       * Child workflow entry show/hide
       */
      _isGoSubProcess () {
        return this.taskType === 'SUB_PROCESS' && this.name
      }
    },
    components: {
      mMr,
      mShell,
      mSubProcess,
      mProcedure,
      mSql,
      mLog,
      mSpark,
      mFlink,
      mPython,
      mDependent,
      mSelectInput,
      mTimeoutAlarm,
      mPriority,
      mWorkerGroups
    }
  }
</script>

<style lang="scss" rel="stylesheet/scss">
  @import "./formModel";
</style>
