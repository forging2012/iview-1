<template>
  <div>
    <Breadcrumb>
      <Breadcrumb-item href="/">首页</Breadcrumb-item>
      <Breadcrumb-item href="#">{{ consts.ALIASES[alias] }}</Breadcrumb-item>
      <Breadcrumb-item>分类列表</Breadcrumb-item>
    </Breadcrumb>
    <List :current="current" :columns="columns" :data="categories.categories.items" :total="categories.categories.total"
          @on-change="handlePageChange">
      <ListHeader>
        <ListOperations>
          <Button class="margin-right-sm" type="primary" @click="handlePost">新增</Button>
        </ListOperations>
        <ListSearch>
          <Form inline @submit.native.prevent="handleSearch">
            <Form-item prop="title">
              <Input type="text" placeholder="请输入标题" v-model="where.title.$like" style="width: 220px;"></Input>
            </Form-item>
            <Form-item>
              <Button type="primary" @click="handleSearch">查询</Button>
            </Form-item>
          </Form>
        </ListSearch>
      </ListHeader>
    </List>
    <Modal width="280" v-model="del.modal" title="请确认" @on-ok="handleDelOk">
      <p>确认删除？</p>
    </Modal>
    <Modal width="500" v-model="formModal" title="新增">
      <Form ref="formValidate" :model="formValidate" :rules="ruleValidate" :label-width="80">
        <Form-item label="标题" prop="title">
          <Row>
            <Col span="20">
              <Input v-model="formValidate.title" placeholder="请输入标题"></Input>
            </Col>
          </Row>
        </Form-item>
        <Form-item label="描述" prop="description">
          <Row>
            <Col span="20">
              <Input v-model="formValidate.description" type="textarea" :rows="3" placeholder="请输入描述"></Input>
            </Col>
          </Row>
        </Form-item>
        <Form-item label="排序" prop="order">
          <Row>
            <Col span="20">
              <InputNumber :min="1" :max="100" v-model="formValidate.order"></InputNumber>
            </Col>
          </Row>
        </Form-item>
      </Form>
      <div slot="footer">
        <Button type="text" size="large" @click="formModal = false">取消</Button>
        <Button type="primary" size="large" @click="handleFormOk">确定</Button>
      </div>
    </Modal>
  </div>
</template>

<script>
  import { mapState } from 'vuex'
  import consts from '@/utils/consts'
  import time from 'apples/libs/time'
  import List, { ListHeader, ListOperations, ListSearch } from '@/components/List'

  export default {
    name: 'list',
    async beforeRouteUpdate (to, from, next) {
      this.categories.categories = {}
      this.alias = to.params.alias
      this.getItems()
      next()
    },
    created () {
      this.categories.categories = {}
      this.alias = this.$route.params.alias
      this.getItems()
    },
    components: {
      List,
      ListHeader,
      ListOperations,
      ListSearch
    },
    data () {
      return {
        consts,
        alias: '',
        formModal: false,
        formValidate: {
          title: '',
          description: '',
          order: 1
        },
        ruleValidate: {
          title: [
            {
              required: true,
              message: '标题不能为空'
            },
            {
              max: 100,
              message: '标题不能多于 100 个字'
            }
          ]
        },
        del: {
          modal: false,
          id: 0
        },
        put: {
          id: 0
        },
        where: {
          title: {
            $like: ''
          }
        },
        current: 1,
        columns: [
          {
            title: 'ID',
            key: 'id',
            width: 80
          },
          {
            title: '标题',
            key: 'title'
          },
          {
            title: '排序',
            key: 'created_at',
            width: 180,
            render (h, params) {
              return h('span', null, params.row.order)
            }
          },
          {
            title: '发布时间',
            key: 'created_at',
            width: 180,
            render (h, params) {
              return h('span', null, time.getDateTime(params.row.created_at + '000'))
            }
          },
          {
            title: '操作',
            key: 'action',
            width: 150,
            render: (h, params) => {
              return h('ButtonGroup', [
                h('Button', {
                  props: {
                    type: 'ghost'
                  },
                  on: {
                    click: () => {
                      this.handlePut(params.row.id)
                    }
                  }
                }, '编辑'),
                h('Button', {
                  props: {
                    type: 'ghost'
                  },
                  on: {
                    click: () => {
                      this.handleDel(params.row.id)
                    }
                  }
                }, '删除')
              ])
            }
          }
        ]
      }
    },
    computed: mapState([
      'categories'
    ]),
    methods: {
      getItems (current = 1) {
        this.current = current

        return this.$store.dispatch('getCategories', {
          query: {
            offset: (current - 1) * consts.PAGE_SIZE,
            limit: consts.PAGE_SIZE,
            where: { ...this.where, alias: this.alias }
          }
        })
      },
      getDetails (id) {
        return this.$store.dispatch('getCategory', { id: this.put.id })
      },
      handlePageChange (current) {
        this.getItems(current)
      },
      handleSearch () {
        this.current = 1
        this.getItems()
      },
      handlePost () {
        this.formModal = true
        this.put.id = 0
        this.$refs.formValidate.resetFields()
      },
      handlePut (id) {
        this.put.id = id
        this.formModal = true
        this.getDetails()
      },
      handleDel (id) {
        this.del.modal = true
        this.del.id = id
      },
      async handleDelOk () {
        await this.$store.dispatch('deleteCategory', {
          id: this.del.id
        })
        this.$Message.success('删除成功！')
        // iView.Spin 的坑，调用 iView.Spin.hide()，500ms 后实例才被销毁
        // await helpers.sleep(500)
        this.getItems()
      },
      handleFormOk () {
        this.$refs.formValidate.validate(async valid => {
          if (valid) {
            const action = this.put.id ? 'putCategory' : 'postCategory'

            await this.$store.dispatch(action, {
              id: this.put.id,
              body: { ...this.formValidate, alias: this.alias }
            })

            this.formModal = false

            this.$Message.success((this.put.id ? '编辑' : '新增') + '成功！')
            !this.put.id && this.$refs.formValidate.resetFields()
            this.getItems()
          }
        })
      }
    },
    watch: {
      'categories.category': {
        handler (newVal) {
          this.formValidate = newVal
        }
      }
    }
  }
</script>
