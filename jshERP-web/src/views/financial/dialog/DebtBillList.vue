<template>
  <a-modal
    :title="title"
    :width="1300"
    :visible="visible"
    @ok="handleOk"
    @cancel="handleCancel"
    cancelText="关闭"
    wrapClassName="ant-modal-cust-warp"
    style="top:5%;height: 100%;overflow-y: hidden">
    <!-- 查询区域 -->
    <div class="table-page-search-wrapper">
      <!-- 搜索区域 -->
      <a-form layout="inline" @keyup.enter.native="searchQuery">
        <a-row :gutter="24">
          <a-col :md="6" :sm="24">
            <a-form-item label="单据编号" :labelCol="{span: 5}" :wrapperCol="{span: 18, offset: 1}">
              <a-input placeholder="请输入单据编号查询" v-model="queryParam.number"></a-input>
            </a-form-item>
          </a-col>
          <a-col :md="6" :sm="24">
            <a-form-item label="商品信息" :labelCol="{span: 5}" :wrapperCol="{span: 18, offset: 1}">
              <a-input placeholder="请输入名称、规格、型号" v-model="queryParam.materialParam"></a-input>
            </a-form-item>
          </a-col>
          <a-col :md="6" :sm="24">
            <a-form-item label="单据日期" :labelCol="labelCol" :wrapperCol="wrapperCol">
              <a-range-picker
                style="width: 100%"
                v-model="queryParam.createTimeRange"
                format="YYYY-MM-DD"
                :placeholder="['开始时间', '结束时间']"
                @change="onDateChange"
                @ok="onDateOk"
              />
            </a-form-item>
          </a-col>
          <span style="float: left;overflow: hidden;" class="table-page-search-submitButtons">
            <a-col :md="6" :sm="24">
              <a-button type="primary" @click="searchQuery">查询</a-button>
              <a-button style="margin-left: 8px" @click="searchReset">重置</a-button>
            </a-col>
          </span>
        </a-row>
      </a-form>
    </div>
    <!-- table区域-begin -->
    <a-table
      bordered
      ref="table"
      size="middle"
      rowKey="id"
      :columns="columns"
      :dataSource="dataSource"
      :pagination="ipagination"
      :loading="loading"
      :rowSelection="{selectedRowKeys: selectedRowKeys, onChange: onSelectChange, type: getType}"
      :customRow="rowAction"
      @change="handleTableChange">
      <span slot="numberCustomRender" slot-scope="text, record">
        <a @click="myHandleDetail(record)">{{record.number}}</a>
      </span>
      <span slot="customTitle">
        实际欠款
        <a-tooltip title="实际欠款=本单欠款-退货单欠款（主要针对存在退货的情况）">
          <a-icon type="question-circle" />
        </a-tooltip>
      </span>
    </a-table>
    <!-- table区域-end -->
    <!-- 表单区域 -->
    <bill-detail ref="modalDetail"></bill-detail>
  </a-modal>
</template>

<script>
  import BillDetail from '../../bill/dialog/BillDetail'
  import { JeecgListMixin } from '@/mixins/JeecgListMixin'
  import { findBillDetailByNumber } from '@/api/api'
  import Vue from 'vue'
  export default {
    name: 'DebtBillList',
    mixins:[JeecgListMixin],
    components: {
      BillDetail
    },
    data () {
      return {
        title: "操作",
        visible: false,
        disableMixinCreated: true,
        selectedRowKeys: [],
        selectionRows: [],
        selectBillRows: [],
        selectBillIds: '',
        queryParam: {
          organId: "",
          materialParam: "",
          number: "",
          type: "",
          subType: "",
          roleType: Vue.ls.get('roleType'),
          status: ""
        },
        labelCol: {
          xs: { span: 24 },
          sm: { span: 8 },
        },
        wrapperCol: {
          xs: { span: 24 },
          sm: { span: 16 },
        },
        // 表头
        columns: [
          { title: '', dataIndex: 'organName',width:120, ellipsis:true},
          {
            title: '单据编号', dataIndex: 'number', width: 120,
            scopedSlots: { customRender: 'numberCustomRender' },
          },
          { title: '商品信息', dataIndex: 'materialsList',width:200, ellipsis:true,
            customRender:function (text,record,index) {
              if(text) {
                return text.replace(",","，");
              }
            }
          },
          { title: '单据日期', dataIndex: 'operTimeStr',width:130},
          { title: '操作员', dataIndex: 'userName',width:70, ellipsis:true},
          { title: '本单欠款', dataIndex: 'needDebt',width:70 },
          { dataIndex: 'realNeedDebt',width:80,
            slots: { title: 'customTitle' }
          },
          { title: '已收欠款', dataIndex: 'finishDebt',width:70 },
          { title: '待收欠款', dataIndex: 'debt',width:70 }
        ],
        url: {
          list: "/depotHead/debtList"
        }
      }
    },
    computed: {
      getType: function () {
        return 'checkbox';
      }
    },
    created() {
    },
    methods: {
      show(organId, type, subType, organType, status) {
        this.queryParam.organId = organId
        this.queryParam.type = type
        this.queryParam.subType = subType
        this.queryParam.status = status
        this.columns[1].title = organType
        if(type === '入库') {
          this.columns[7].title = '已付欠款'
          this.columns[8].title = '待付欠款'
        } else if(type === '出库') {
          this.columns[7].title = '已收欠款'
          this.columns[8].title = '待收欠款'
        }
        this.model = Object.assign({}, {});
        this.visible = true;
        this.ipagination.pageSize = 100
        this.ipagination.pageSizeOptions = ['100', '200', '300']
        this.loadData(1)
      },
      myHandleDetail(record) {
        findBillDetailByNumber({ number: record.number }).then((res) => {
          if (res && res.code === 200) {
            let type = res.data.depotHeadType
            type = type.replace('其它','')
            this.handleDetail(res.data, type)
          }
        })
      },
      close () {
        this.$emit('close');
        this.visible = false;
      },
      handleCancel () {
        this.close()
      },
      onSelectChange(selectedRowKeys, selectionRows) {
        this.selectedRowKeys = selectedRowKeys;
        this.selectionRows = selectionRows;
      },
      handleOk () {
        this.getSelectBillRows();
        this.$emit('ok', this.selectBillRows);
        this.selectedRowKeys = []
        this.selectBillRows = []
        this.close();
      },
      onDateChange: function (value, dateString) {
        this.queryParam.beginTime=dateString[0];
        this.queryParam.endTime=dateString[1];
      },
      onDateOk(value) {
        console.log(value);
      },
      searchReset() {
        this.queryParam = {
          type: this.queryParam.type,
          subType: this.queryParam.subType
        }
        this.loadData(1);
      },
      getSelectBillRows() {
        let dataSource = this.dataSource;
        let billIds = "";
        this.selectBillRows = []
        for (let i = 0, len = dataSource.length; i < len; i++) {
          if (this.selectedRowKeys.includes(dataSource[i].id)) {
            this.selectBillRows.push(dataSource[i]);
            billIds = billIds + "," + dataSource[i].id
          }
        }
        this.selectBillIds = billIds.substring(1);
      },
      rowAction(record, index) {
        return {
          on: {
            click: () => {
              let arr = []
              arr.push(record.id)
              this.selectedRowKeys = arr
            },
            dblclick: () => {
              let arr = []
              arr.push(record.id)
              this.selectedRowKeys = arr
              this.handleOk()
            }
          }
        }
      }
    }
  }
</script>

<style scoped>

</style>