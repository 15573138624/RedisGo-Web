<template>
  <div ref="data_view">
    <a-row type="flex" justify="center">
      <a-col span="7">
        <a-input-group compact style="display: flex">
          <a-select v-if="info_data !== undefined" style="width: 110px" :value="redis_db" @change="change_db">
            <a-select-option v-for="item in dbs" :value="item.value" :key="item.value">{{item.label}}</a-select-option>
          </a-select>
          <a-input-search style="width: 200px" placeholder="请输入要搜索的Key"
            v-model="search_key"
            @search="search_keys(true, true)"
          />
          <a-button icon="plus" @click="add_key_click">添加</a-button>
        </a-input-group>

        <div style="margin-top: 10px"/>

        <a-list bordered :dataSource="keys" style="max-height: 80vh; overflow: auto; background-color: white">
          <a-list-item slot="renderItem" slot-scope="item" @click="click_item(item)" class="rs-data-menu" style="overflow: hidden; display: flex; align-items: center">
            <a-list-item-meta style="margin-left: 10px">
              <a slot="title">
                <a-tooltip :key="item['name']" :title="item['type']+' 类型: '+item['name']" style="display: flex">
                  <a-tag v-if="item['type']==='string'" class="tagstyle" color="green">String</a-tag>
                  <a-tag v-if="item['type']==='list'" class="tagstyle" color="blue">List</a-tag>
                  <a-tag v-if="item['type']==='set'" class="tagstyle" color="pink">Set</a-tag>
                  <a-tag v-if="item['type']==='zset'" class="tagstyle" color="red">ZSet</a-tag>
                  <a-tag v-if="item['type']==='hash'" class="tagstyle" color="cyan">Hash</a-tag>
                  <div style="overflow: hidden;text-overflow: ellipsis;white-space: nowrap;">{{item['name'].length > 23 ? `${item['name'].slice(0, 23)}...` : item['name']}}</div>
                </a-tooltip>
              </a>
            </a-list-item-meta>
            <div slot="actions">Len: {{item['len']}} | TTL: {{formatSeconds(item['ttl'] / 1000000000)}}</div>
          </a-list-item>
        </a-list>
        <div style="text-align: right; margin-top: 10px">
          <a-button-group>
            <a-button v-if="pre_key_flag" @click="search_keys(false, false)" :loading="left_loading"> <a-icon type="left" /> Prev</a-button>
            <a-button v-if="next_key_flag" @click="search_keys(false, true)" :loading="left_loading"> Next<a-icon type="right"/></a-button>
          </a-button-group>
        </div>
      </a-col>

      <a-col offset="1" :md="16" :xl="16" :xxl="12">
        <template v-if="temp_key_item.type !== 'None'" >
          <div style="display: flex;align-items: center;">
            <a-input style="width: 268px" placeholder="名称" v-model="temp_key_item.name" @pressEnter="rename_key">
              <span slot="addonBefore">{{temp_key_item.type.toUpperCase()}}</span>
              <a-icon type="check" slot="suffix" @click="rename_key"/>
            </a-input>

            <a-input style="width: 136px; margin-left: 8px" placeholder="过期时间" v-model="temp_key_item.ttl" @pressEnter="update_ttl">
              <span slot="addonBefore">TTL</span>
              <a-icon type="check" slot="suffix" @click="update_ttl(temp_key_item.ttl)"/>
            </a-input>

            <a-button-group style="margin-left: 8px;margin-right: 8px;">
              <a-tooltip title="刷新">
                <a-button icon="sync" @click="refresh" style="color: green"/>
              </a-tooltip>
              <a-tooltip title="删除Key">
                <a-popconfirm
                    title="确认删除吗?" ok-text="确认" cancel-text="取消"
                    @confirm="rm_key"
                >
                    <a-button icon="delete" style="color: red"/>
                </a-popconfirm>
              </a-tooltip>
            </a-button-group>

            <span style="display: flex;align-items: center;margin-left: auto;">
              <template v-if="temp_key_item.type === 'string'">
                <a-button-group>
                  <a-tooltip title="以初始文字形式展示">
                    <a-button @click="show_text" style="width: 33%">Text</a-button>
                  </a-tooltip>
                  <a-tooltip title="以Json形式展示">
                    <a-button @click="show_json" style="width: 33%">Json</a-button>
                  </a-tooltip>
                  <a-tooltip title="以json压缩形式展示">
                    <a-button @click="show_zip" style="width: 33%">Zip</a-button>
                  </a-tooltip>
                </a-button-group>
                <a-button v-if="!edit_mode" style="margin-left: 4px" @click="edit_value"><a-icon type="edit"/>编辑</a-button>
                <a-button-group v-else style="margin-left: 4px">
                  <a-button type="primary" @click="comform_edit">确认</a-button>
                  <a-button @click="cancel_edit">取消</a-button>
                </a-button-group>
              </template>
              <template v-if="['hash', 'set', 'zset'].includes(temp_key_item.type)">
                <a-input-search style="flex: 1" placeholder="请输入要搜索的Key"
                                v-model="value_search_key"
                                @search="value_search_key_value(true, true)"
                />
              </template>
              <template v-if="temp_key_item.type !== 'string'">
                <a-button style="margin-left: 4px" @click="append_key_value_modal=true"><a-icon type="plus"/>添加元素</a-button>
              </template>
            </span>
          </div>
        </template>


        <div style="margin-top: 10px"/>

        <div style="max-height: 80vh; overflow: auto; background-color: white">
          <div class="spin-content" v-show="present_spin">
            <a-spin :spinning="present_spin"></a-spin>
          </div>
          <div v-show="!present_spin">
            <div v-if="temp_key_item.type === 'string'">
              <div v-if="!edit_mode" >
                <json-view v-if="json_view_flag" :data="temp_key_item.key_value" />
                <pre v-else>{{temp_key_item.key_value}}</pre>
              </div>
              <a-textarea v-else v-model="temp_key_item.key_value" :rows="60" style="max-height: 80vh; overflow: auto;" placeholder="暂无内容" />
            </div>
            <div v-if="['list', 'hash', 'set', 'zset'].includes(temp_key_item.type)">
              <a-list bordered :dataSource="temp_key_item.key_value">
                <a-list-item slot="renderItem" slot-scope="item, index">
                  <a-list-item-meta>
                    <template v-if="temp_key_item.type === 'list' || temp_key_item.type === 'set'">
                      <div slot="title">
                        <a-textarea v-if="item[2]" v-model="item[1]" :rows="cal_textarea_lines(item[1])"/>
                        <span v-else>{{item[1]}}</span>
                      </div>
                    </template>
                    <template v-else-if="temp_key_item.type === 'hash'">
                      <span slot="title">{{item[0]}}</span>
                      <div slot="description">
                        <a-textarea v-if="item[2]" v-model="item[1]" :rows="cal_textarea_lines(item[1])"/>
                        <span v-else>{{item[1]}}</span>
                      </div>
                    </template>
                    <template v-else-if="temp_key_item.type === 'zset'">
                      <template slot="title">
                        <a-input v-if="item[2]" v-model="item[1]" style="width: 80px"/>
                        <span v-else>{{item[1]}}</span>
                        <a-divider type="vertical"/> {{item[0]}}
                      </template>
                    </template>
                  </a-list-item-meta>
                  <div slot="actions">
                    <template v-if="item[2]">
                      <a @click="conform_edit_item_value(index)">确认</a> <a-divider type="vertical"/>
                      <a @click="cancel_edit_item_value(index)" style="color: lightsalmon">取消</a>
                    </template>
                    <template v-else>
                      <a @click="temp_key_item.type !== 'zset' ? format_json(item[1]) : format_json(item[0])">Json</a>
                      <a-divider type="vertical"/>
                      <a @click="edit_item_value(index)">编辑</a> <a-divider type="vertical"/>
                      <a-popconfirm
                          title="确认删除吗?" ok-text="确认" cancel-text="取消"
                          @confirm="delete_item_value(index)"
                      >
                        <a style="color: lightsalmon">删除</a>
                      </a-popconfirm>
                    </template>
                  </div>
                </a-list-item>
              </a-list>
            </div>
          </div>
        </div>
        <div style="text-align: right; margin-top: 10px">
          <a-button-group v-if="temp_key_item.type === 'hash' || temp_key_item.type === 'set'">
            <a-button v-if="pre_value_page_flag" @click="value_search_key_value(false, false)" :loading="right_loading"> <a-icon type="left" /> Prev</a-button>
            <a-button v-if="next_value_page_flag" @click="value_search_key_value(false, true)" :loading="right_loading"> Next <a-icon type="right"/></a-button>
          </a-button-group>
          <a-pagination v-if="temp_key_item.type === 'list'" v-model="list_page"  @change="list_page_click" :defaultPageSize="value_count" :total="temp_key_item.len" />
        </div>
      </a-col>
    </a-row>
    <a-modal v-model="showJsonModal" :footer="null" :destroyOnClose="true" width="50vw">
      <json-view :data="jsonDataModal" style="margin-top: 20px; overflow: auto; max-height: 72vh"/>
    </a-modal>
    <AddDataModal ref="add_data_modal" :search_keys="search_keys"/>
    <a-modal v-model="append_key_value_modal" okText="确认" cancelText="取消" @ok="append_key_value_commit" :destroyOnClose="true" width="500px">
      <span slot="title">插入数据 <a-divider type="vertical"/> {{temp_key_item.name}}</span>
      <a-input-group v-if="temp_key_item.type==='list'" compact>
        <span style="font-size: 16px; width: 80px">插入位置: </span>
        <a-input-number v-model="add_redis_key.pos" placeholder="整数" :precision="0" :min="-1" :max="1000000000" style="width: 360px"/>
      </a-input-group>
      <a-input-group v-if="temp_key_item.type==='zset'" style="margin-top: 8px" compact>
        <span style="font-size: 16px; width: 80px">数据分数: </span>
        <a-input-number v-model="add_redis_key.score" placeholder="作为排序依据" :precision="8" :min="0" :max="1000000000" style="width: 360px"/>
      </a-input-group>
      <a-input-group v-if="temp_key_item.type==='hash'" style="margin-top: 8px" compact>
        <span style="font-size: 16px; width: 80px">哈希键值: </span>
        <a-input v-model="add_redis_key.key" style="width: 360px"/>
      </a-input-group>
      <a-input-group style="margin-top: 8px" compact>
        <span style="font-size: 16px; width: 80px">数据详情: </span>
        <a-textarea v-model="add_redis_key.value" :rows="6" style="width: 360px"/>
      </a-input-group>
      <a-input-group style="margin-top: 8px" compact>
        <span style="font-size: 16px; width: 80px"></span>
        <a-checkbox v-model="add_redis_key.zip_json" @change="try_zip_json">尝试压缩Json文本</a-checkbox>
      </a-input-group>
    </a-modal>
  </div>
</template>

<script>
// import moment from 'moment'
import C from '@/config'
import U from "@/utils"
import jsonView from 'vue-json-views'
import AddDataModal from "@/components/AddDataModal"
import {mapState, mapGetters, mapMutations, mapActions} from 'vuex'

export default {
  name: 'RedisData',
  data() {
    return {
      // eslint-disable-next-line no-console
      log: console.log,
      formatSeconds: U.formatSeconds,
      row_width: [7, 12],
      redis_db: 0,
      search_key: "",
      left_loading: false,
      right_loading: false,
      key_cursors: [0],
      key_count: 20,
      keys: [],
      present_mode: 'Text',
      value_cursors: [0],
      value_search_key: "",
      value_count: 20,
      list_page: 1,
      present_spin: false,
      json_view_flag: false,
      jsonDataModal: null,
      showJsonModal: false,
      edit_mode: false,
      origin_key_item: {name: "", type: 'None', len: 0, ttl: -1, key_value: '暂无内容'},
      temp_key_item: {name: "", type: 'None', len: 0, ttl: -1, key_value: '暂无内容'},
      append_key_value_modal: false,
      add_redis_key: {pos: -1, score: 0, key: '', value: '', zip_json: false}
    }
  },
  components: { jsonView, AddDataModal },
  watch: {
    redis_id(val) {
      if (val !== '') {
        this.getRedisInfo()
        this.search_keys()
      }
    }
  },
  computed: {
    ...mapState(['redis_id']),
    ...mapGetters(['info_data']),
    dbs: function () {
      let all_db = []
      for (let i = 0; i < 16; i++) {
        all_db.push({'value': i, 'label': `DB${i} (${this.format_db_nums(this.info_data['db'+i])})`})
      }
      return all_db
    },
    pre_key_flag: function () {
      return this.key_cursors.length > 2
    },
    next_key_flag: function () {
      return this.keys.length >= this.key_count
    },
    pre_value_page_flag: function () {
      return this.value_cursors.length > 2
    },
    next_value_page_flag: function () {
      return (this.temp_key_item.key_value.length >= this.value_count)
          && (this.value_cursors[this.value_cursors.length - 1] !== 0)
    },
  },
  methods: {
    ...mapMutations(['setRedisInfo']),
    ...mapActions(["getRedisInfo"]),
    cal_textarea_lines(content) {
      return parseInt(content.length / 100.0) + 1
    },
    format_db_nums(str) {
      if (str === undefined) return 0
      if (str === "") return 0
      let s = str.split(',')[0]
      return s.split('=')[1]
    },
    async change_db(value) {
      this.redis_db = value
      const body = await C.myaxios.post(`data/common?method=select_db&id=${this.redis_id}&db=${this.redis_db}`)
      if (body.status === 200 && body.data && body.data.code === 0) {
        if (body.data.data === "OK") {
          this.$message.success(`成功切换至DB${value}`)
          await this.search_keys()
        }
      }
    },
    async search_keys(reset=true, search_next=true) {
      this.left_loading = true
      if (reset) this.key_cursors = [0]
      if (!search_next) {
        this.key_cursors.pop()
        this.key_cursors.pop()
      }
      let cursor = this.key_cursors[this.key_cursors.length - 1]
      let match = this.search_key !== '' ? `*${this.search_key}*` : '*'
      const body = await C.myaxios.post(`data/common?method=get_keys&id=${this.redis_id}`, {cursor, match, count: this.key_count})
      this.left_loading = false
      if (body.status === 200 && body.data && body.data.code === 0) {
        let data = body.data.data
        this.key_cursors.push(data.cursor)
        this.keys = data.keys === null? [] : data.keys
      }
    },
    async edit_value() {
      this.edit_mode = true
      if (typeof this.temp_key_item.key_value === "object") {
        this.temp_key_item.key_value = JSON.stringify(this.temp_key_item.key_value, null, 4)
      }
    },
    add_key_click() {
      this.$refs.add_data_modal.show()
    },
    async rm_key() {
      const body = await C.myaxios.post(`data/common?method=rm_key&id=${this.redis_id}&key=${this.origin_key_item.name}`)
      if (body.status === 200 && body.data && body.data.code === 0) {
        if (body.data.data === 1) {
          this.$message.success("删除成功")
          await this.search_keys()
          await this.get_info()
          this.origin_key_item.name = ''
          this.origin_key_item.ttl = -1
          this.origin_key_item.key_value = '暂无内容'
          this.temp_key_item.name = ''
          this.temp_key_item.ttl = -1
          this.temp_key_item.key_value = '暂无内容'
        }
      }
    },
    cancel_edit() {
      this.edit_mode = false
      if (this.present_mode === 'Json') {
        this.temp_key_item.key_value = JSON.parse(this.origin_key_item.key_value)
      }
    },
    async comform_edit() {
      await this.set_string_value(this.temp_key_item.key_value)
    },
    async refresh() {
      await this.get_ttl()
      await this.getRedisInfo()
      await this.get_key_value(this.origin_key_item.name, this.origin_key_item.type)
      if (this.present_mode === 'Json') {
        this.temp_key_item.key_value = JSON.parse(this.temp_key_item.key_value)
      }
    },
    async rename_key() {
      const body = await C.myaxios.post(`data/common?method=rename&id=${this.redis_id}&key=${this.origin_key_item.name}`, {new_name: this.temp_key_item.name})
      if (body.status === 200 && body.data && body.data.code === 0) {
        if (body.data.data === true || body.data.data === "OK") {
          this.$message.success('重命名成功')
          this.origin_key_item.name = this.temp_key_item.name
          await this.getRedisInfo()
          await this.search_keys()
        } else {
          this.$message.error('重命名失败: ' + body.data.data)
        }
      }
    },
    async update_ttl(value) {
      const body = await C.myaxios.post(`data/common?method=update_ttl&id=${this.redis_id}&key=${this.origin_key_item.name}&ttl=${value}`)
      if (body.status === 200 && body.data && body.data.code === 0) {
        if (body.data.data === true || body.data.data === "OK") {
          this.$message.success('更新超时时间成功')
          await this.search_keys()
          await this.getRedisInfo()
        }
      }
    },
    async get_ttl() {
      const body = await C.myaxios.post(`data/common?method=get_ttl&id=${this.redis_id}&key=${this.origin_key_item.name}`)
      if (body.status === 200 && body.data && body.data.code === 0) {
        this.origin_key_item.ttl = parseInt(body.data.data / 1000000000)
        this.temp_key_item.ttl = parseInt(body.data.data / 1000000000)
      }
    },
    async click_item(item) {
      this.json_view_flag = false
      const [value1,value2] = await this.get_retyrn_key_value(item.name, item.type)
      this.origin_key_item = {
        ttl: parseInt(item.ttl / 1000000000),
        type: item.type,
        name: item.name,
        len: item.len,
        key_value: value1
      }

      this.temp_key_item = {
        ttl: parseInt(item.ttl / 1000000000),
        type: item.type,
        name: item.name,
        len: item.len,
        key_value: value2
      }

      // this.origin_key_item.ttl = parseInt(item.ttl / 1000000000)
      // this.origin_key_item.type = item.type
      // this.origin_key_item.name = item.name
      // this.origin_key_item.len = item.len

      // this.temp_key_item.ttl = parseInt(item.ttl / 1000000000)
      // this.temp_key_item.type = item.type
      // this.temp_key_item.name = item.name
      // this.temp_key_item.len = item.len

      this.value_search_key = ""
      // await this.get_key_value(item.name, item.type)
      this.present_mode = 'Text'
    },
    async value_search_key_value(reset=true, search_next=true) {
      this.right_loading = true
      let match = this.value_search_key === '' ? '*' : `*${this.value_search_key}*`
      await this.get_key_value(this.temp_key_item.name, this.temp_key_item.type, match, reset, search_next)
      this.right_loading = false
    },
    async list_page_click(page) {
      this.list_page = page
      await this.get_key_value(this.temp_key_item.name, this.temp_key_item.type, '', false)
    },
    async get_retyrn_key_value(name, type, match='*', reset=true, search_next=true) {
      this.present_spin = true
      if (type === 'hash' || type === 'set' || type === 'zset') {
        if (reset) this.value_cursors = [0]
        if (!search_next) {
          this.value_cursors.pop()
          this.value_cursors.pop()
        }
        let cursor = this.value_cursors[this.value_cursors.length - 1]
        const body = await C.myaxios.post(`data/${type}?method=get&id=${this.redis_id}&key=${name}`, {cursor, match, count: this.value_count})
        if (body.status === 200 && body.data && body.data.code === 0) {
          this.present_spin = false
          let data = body.data.data
          if(data.keys){
            this.value_cursors.push(data.cursor)
            let valueData1 = []
            let valueData2 = []
            if (type === 'hash' || type === 'zset') {
              for (let i = 0; i < data.keys.length - 1; i += 2) {
                valueData1.push([data.keys[i], data.keys[i+1], false])
                valueData2.push([data.keys[i], data.keys[i+1], false])
              }
            } else if (type === 'set') {
              for (let i = 0; i < data.keys.length; i += 1) {
                valueData1.push(['', data.keys[i], false])
                valueData2.push(['', data.keys[i], false])
              }
            }
            return [valueData1,valueData2]
          }else{
            return [[],[]]
          }
        }
      } else if (type === 'list') {
        if (reset) this.list_page = 1
        let start = (this.list_page - 1) * this.value_count
        let end = start + this.value_count - 1
        const body = await C.myaxios.post(`data/${type}?method=get&id=${this.redis_id}&key=${name}`, {start, end})
        if (body.status === 200 && body.data && body.data.code === 0) {
          this.present_spin = false
          let data = body.data.data
           if(data.keys){
             let listData1 = []
             let listData2 = []
             for (let i = 0; i < data.length; i += 1) {
               listData1.push(['', data[i], false])
               listData2.push(['', data[i], false])
             }
             return [listData1,listData2]
           }else{
             return [[],[]]
           }
        }
      } else if (type === 'string') {
        const body = await C.myaxios.post(`data/${type}?method=get&id=${this.redis_id}&key=${name}`)
        if (body.status === 200 && body.data && body.data.code === 0) {
          this.present_spin = false
          return [body.data.data || [], body.data.data || []]
        }
      }
    },
    async get_key_value(name, type, match='*', reset=true, search_next=true) {
      this.present_spin = true
      if (type === 'hash' || type === 'set' || type === 'zset') {
        if (reset) this.value_cursors = [0]
        if (!search_next) {
          this.value_cursors.pop()
          this.value_cursors.pop()
        }
        let cursor = this.value_cursors[this.value_cursors.length - 1]
        const body = await C.myaxios.post(`data/${type}?method=get&id=${this.redis_id}&key=${name}`, {cursor, match, count: this.value_count})
        if (body.status === 200 && body.data && body.data.code === 0) {
          this.present_spin = false
          let data = body.data.data
          if(data.keys){
            this.value_cursors.push(data.cursor)
            let valueData1 = []
            let valueData2 = []
            if (type === 'hash' || type === 'zset') {
              for (let i = 0; i < data.keys.length - 1; i += 2) {
                valueData1.push([data.keys[i], data.keys[i+1], false])
                valueData2.push([data.keys[i], data.keys[i+1], false])
              }
            } else if (type === 'set') {
              for (let i = 0; i < data.keys.length; i += 1) {
                valueData1.push(['', data.keys[i], false])
                valueData2.push(['', data.keys[i], false])
              }
            }
            this.origin_key_item.key_value = valueData1
            this.temp_key_item.key_value = valueData2
          }else{
            this.origin_key_item.key_value = []
            this.temp_key_item.key_value = []
          }
        }
      } else if (type === 'list') {
        if (reset) this.list_page = 1
        let start = (this.list_page - 1) * this.value_count
        let end = start + this.value_count - 1
        const body = await C.myaxios.post(`data/${type}?method=get&id=${this.redis_id}&key=${name}`, {start, end})
        if (body.status === 200 && body.data && body.data.code === 0) {
          this.present_spin = false
          let data = body.data.data
           if(data.keys){
             let listData1 = []
             let listData2 = []
             for (let i = 0; i < data.length; i += 1) {
               listData1.push(['', data[i], false])
               listData2.push(['', data[i], false])
             }
             this.origin_key_item.key_value = listData1
             this.temp_key_item.key_value = listData2
           }else{
             this.origin_key_item.key_value = []
             this.temp_key_item.key_value = []
           }
        }
      } else if (type === 'string') {
        const body = await C.myaxios.post(`data/${type}?method=get&id=${this.redis_id}&key=${name}`)
        if (body.status === 200 && body.data && body.data.code === 0) {
          this.present_spin = false
          this.origin_key_item.key_value = body.data.data === null? [] : body.data.data
          this.temp_key_item.key_value = body.data.data === null? [] : body.data.data
        }
      }
    },
    format_json(json_data) {
      let jsonData = U.parse_json(json_data)
      if (jsonData !== null) {
        this.jsonDataModal = jsonData
        this.showJsonModal = true
      } else {
        this.$message.error('不支持该类型数据的Json展示')
      }
    },
    show_json() {
      if (!this.edit_mode) {
        if (typeof this.temp_key_item.key_value !== "object") {
          let jsonData = U.parse_json(this.temp_key_item.key_value)
          if (jsonData !== null) {
            this.temp_key_item.key_value = jsonData
            this.present_mode = 'Json'
          } else {
            this.$message.error('不支持该类型数据的Json展示')
            return
          }
        }
        this.json_view_flag = true
      } else {
        if (typeof this.temp_key_item.key_value === "object") {
          this.temp_key_item.key_value = JSON.stringify(this.temp_key_item.key_value, null, 4)
          this.present_mode = 'Json'
        } else {
          let jsonData = U.parse_json(this.temp_key_item.key_value)
          if (jsonData !== null) {
            this.temp_key_item.key_value = JSON.stringify(jsonData, null, 4)
            this.present_mode = 'Json'
          } else {
            this.$message.error('不支持该类型数据的Json展示')
          }
        }
      }
    },
    show_zip() {
      this.json_view_flag = false
      if (typeof this.temp_key_item.key_value === "object") {
        this.temp_key_item.key_value = JSON.stringify(this.temp_key_item.key_value)
        this.present_mode = 'Zip'
      } else {
        let jsonData = U.parse_json(this.temp_key_item.key_value)
        if (jsonData !== null) {
          this.temp_key_item.key_value = JSON.stringify(jsonData)
          this.present_mode = 'Zip'
        } else {
          this.$message.error('不支持该类型数据的Json展示')
        }
      }
    },
    show_text() {
      this.json_view_flag = false
      this.temp_key_item.key_value = this.origin_key_item.key_value
      this.present_mode = 'Text'
    },
    async set_string_value(value) {
      const body = await C.myaxios.post(`data/string?method=set&id=${this.redis_id}&key=${this.temp_key_item.name}`, {value})
      if (body.status === 200 && body.data && body.data.code === 0) {
        if (body.data.data === "OK") {
          this.$message.success("修改成功")
          this.edit_mode = false
          if (this.present_mode === 'Json') {
            this.temp_key_item.key_value = JSON.parse(value)
          }
          this.origin_key_item.key_value = value
        }
      }
    },
    async delete_item_value(index) {
      let body = null
      let common = `method=delete&&id=${this.redis_id}&key=${this.temp_key_item.name}`
      if (this.temp_key_item.type === 'list') {
        let list_pos = (this.list_page - 1) * this.value_count + index
        body = await C.myaxios.post(`data/list?${common}`, {pos: list_pos})
      } else if (this.temp_key_item.type === 'hash') {
        let hash_key = this.origin_key_item.key_value[index][0]
        body = await C.myaxios.post(`data/hash?${common}`, {hash_key})
      } else if (this.temp_key_item.type === 'set') {
        let set_key = this.origin_key_item.key_value[index][1]
        body = await C.myaxios.post(`data/set?${common}`, {set_key})
      } else if (this.temp_key_item.type === 'zset') {
        let zset_key = this.origin_key_item.key_value[index][0]
        body = await C.myaxios.post(`data/zset?${common}`, {zset_key})
      }

      if (body.status === 200 && body.data && body.data.code === 0) {
        this.$message.success('删除成功')
        this.temp_key_item.key_value.splice(index, 1)
        this.origin_key_item.key_value.splice(index, 1)
        await this.getRedisInfo()
      }
    },
    async edit_item_value(index) {
      this.temp_key_item.key_value[index].splice(2, 1, true)
    },
    async conform_edit_item_value(index) {
      let new_value = this.temp_key_item.key_value[index][1]
      let common = `method=set&&id=${this.redis_id}&key=${this.temp_key_item.name}`
      let body = null
      if (this.temp_key_item.type === 'list') {
        let list_pos = (this.list_page - 1) * this.value_count + index
        body = await C.myaxios.post(`data/list?${common}`, {pos: list_pos, value: new_value})
      } else if (this.temp_key_item.type === 'hash') {
        let hash_key = this.origin_key_item.key_value[index][0]
        body = await C.myaxios.post(`data/hash?${common}`, {hash_key, value: new_value})
      } else if (this.temp_key_item.type === 'set') {
        let set_key = this.origin_key_item.key_value[index][1]
        body = await C.myaxios.post(`data/set?${common}`, {set_key, value: new_value})
      } else if (this.temp_key_item.type === 'zset') {
        let zset_key = this.origin_key_item.key_value[index][0]
        let reg=/^[0-9]+.?[0-9]*$/
        if (!reg.test(new_value)) {
          this.$message.warning('Redis的ZSET类型的值只支持数字')
          return
        }
        body = await C.myaxios.post(`data/zset?${common}`, {zset_key, zvalue: parseFloat(new_value)})
      }

      if (body.status === 200 && body.data && body.data.code === 0) {
        this.$message.success('修改成功')
        this.temp_key_item.key_value[index].splice(2, 1, false)
        this.origin_key_item.key_value[index].splice(1, 1, this.temp_key_item.key_value[index][1])
      } else {
        this.$message.error(body.data.data)
      }
    },
    async cancel_edit_item_value(index) {
      this.temp_key_item.key_value[index].splice(2, 1, false)
      this.temp_key_item.key_value[index].splice(1, 1, this.origin_key_item.key_value[index][1])
    },
    async append_key_value_commit() {
      let data = this.add_redis_key
      let body = undefined
      let success = false
      let common = `id=${this.redis_id}&key=${this.temp_key_item.name}`
      if (this.temp_key_item.type === 'list') {
        body = await C.myaxios.post( `data/list?method=push&${common}`, {pos: data.pos, value: data.value})
        if (body.status === 200 && body.data && body.data.code === 0 && body.data.data > 0) {
          success = true
        } else {
          body.data.data = '执行失败, 请检查插入位置是否非法或者键已被删除'
        }
      } else if (this.temp_key_item.type === 'hash') {
        body = await C.myaxios.post(`data/hash?method=add&${common}`, {hash_key: data.key, value: data.value})
        if (body.status === 200 && body.data && body.data.code === 0 && body.data.data) {
          success = true
        } else {
          body.data.data = '执行失败, 请检查是否键名冲突'
        }
      } else if (this.temp_key_item.type === 'set') {
        body = await C.myaxios.post(`data/set?method=add&${common}`, {value: data.value})
        if (body.status === 200 && body.data && body.data.code === 0 && body.data.data > 0) {
          success = true
        } else {
          body.data.data = '执行失败, 请检查是否键名冲突'
        }
      } else if (this.temp_key_item.type === 'zset') {
        body = await C.myaxios.post(`data/zset?method=add&${common}`, {zset_key: data.value, zvalue: data.score})
        if (body.status === 200 && body.data && body.data.code === 0) {
          success = true
          if (body.data.data === 0) {
            this.$message.success('添加的数据已经存在, 以为你更新数据分数值')
          }
        } else {
          body.data.data = '执行失败, 请检查是否键名冲突'
        }
      }
      if (success) {
        this.$message.success('执行成功')
        this.append_key_value_modal = false
        await this.get_key_value(this.temp_key_item.name, this.temp_key_item.type, '', false)
        await this.getRedisInfo()
        await this.search_keys()
      } else {
        this.$message.error(body.data.data)
      }
    },
    try_zip_json() {
      if (this.add_redis_key.zip_json) {
        let jsonData = U.parse_json(this.add_redis_key.value)
        if (jsonData !== null) {
          this.add_redis_key.value_tmp = this.add_redis_key.value
          this.add_redis_key.value = JSON.stringify(jsonData)
        } else {
          this.$message.error('不支持该类型数据的Json展示')
          this.add_redis_key.zip_json = false
        }
      } else {
        this.add_redis_key.value = this.add_redis_key.value_tmp
      }
    }
  },
  mounted() {
    if (this.$refs.data_view.width <= 900) {
      this.row_width = [8, 15]
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.tagstyle {
  text-align: center;
}
.spin-content {
  text-align: center;
  background: rgba(0, 0, 0, 0.05);
  border-radius: 4px;
  margin-bottom: 20px;
  padding: 30px 50px;
  margin: 20px 0;
}
.ant-list-item-meta-description{
  word-break: break-all;
}
pre {padding: 5px; margin: 5px; white-space: pre-wrap; word-wrap: break-word;}
.string { color: green; }
.number { color: darkorange; }
.boolean { color: blue; }
.null { color: magenta; }
.key { color: red; }
.rs-data-menu .ant-list-item-meta {
  display: flex;
  flex: 1;
  overflow: hidden;
}
.rs-data-menu .ant-list-item-meta .ant-list-item-meta-content {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.rs-data-menu /deep/ .ant-list-item-action {
  margin-left: 10px;
}
.rs-data-menu /deep/ ul li {
  padding: 0;
}

.rs-data-menu h4 {
  margin-bottom: 0px;
}
</style>
