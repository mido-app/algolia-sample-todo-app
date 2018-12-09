<template>
  <div class="p-2">
    <b-form-row>
      <b-col>
        <b-form-input type="text" v-model="query" />
      </b-col>
      <b-col cols="2">
        <b-button block @click="searchTodo">検索</b-button>
      </b-col>
    </b-form-row>
    <b-button class="my-2" variant="primary"  block @click="openRegisterModal">TODOを追加</b-button>

    <!-- TODO一覧 -->
    <b-card class="my-2"
            v-for="todo in highlightedTodoList" 
            :key="todo.objectID"
            :title="todo.title">
      <p v-html="todo.description"></p>
      <b-button v-if="!todo.done" variant="primary" @click="makeTodoDone(todo)">完了</b-button>
      <b-button v-else disabled>完了済</b-button>
      <b-button @click="openUpdateModal(todo)">編集</b-button>
      <b-button @click="deleteTodo(todo)">削除</b-button>
    </b-card>

    <!-- 登録用ダイアログ -->
    <b-modal v-model="registerModalIsVisible"
             @ok="registerTodo"
             @calcel="clearInput">
      <b-form-group label="Title" label-for="title">
       <b-form-input id="title" type="text" v-model="todoInput.title" />
      </b-form-group>
      <b-form-group label="Description" label-for="description">
       <b-form-input id="description" type="text" v-model="todoInput.description" />
      </b-form-group>
    </b-modal>

    <!-- 更新用ダイアログ -->
    <b-modal v-model="updateModalIsVisible"
             @ok="updateTodo"
             @calcel="clearInput">
      <b-form-group label="Title" label-for="title">
       <b-form-input id="title" type="text" v-model="todoInput.title" />
      </b-form-group>
      <b-form-group label="Description" label-for="description">
       <b-form-input id="description" type="text" v-model="todoInput.description" />
      </b-form-group>
    </b-modal>
  </div>
</template>

<script>
import * as algoliasearch from 'algoliasearch'
import config from '~/algolia.config.js'

const client = algoliasearch(config.appId, config.apiKey)
const index = client.initIndex('todo')

index.setSettings({
  attributesToHighlight: [
    'title',
    'description'
  ],
  highlightPreTag: '<em class="search-highlight">',
  highlightPostTag: '</em>'
})

export default {
  async asyncData () {
    let searchResult = await index.search({ query: '' })
    return {
      todoList: searchResult.hits
    }
  },
  data () {
    return {
      query: '',
      todoList: [],
      todoInput: {
        title: '',
        description: '',
        done: false
      },
      registerModalIsVisible: false,
      updateModalIsVisible: false,
      updateTargetTodo: null
    }
  },
  methods: {
    clearInput () {
      this.todoInput.title = ''
      this.todoInput.description = '',
      this.todoInput.done = false
    },
    openRegisterModal () {
      this.registerModalIsVisible = true
    },
    async registerTodo () {
      let todo = Object.assign({}, this.todoInput)
      let content = await index.addObject(todo)
      todo.objectID = content.objectID
      this.todoList.push(todo)
      this.clearInput()
    },
    openUpdateModal (todo) {
      this.todoInput.title = todo.title
      this.todoInput.description = todo.description
      this.updateTargetTodo = todo
      this.updateModalIsVisible = true
    },
    async updateTodo () {      
      this.updateTargetTodo.title = this.todoInput.title
      this.updateTargetTodo.description = this.todoInput.description
      await index.saveObject(this.updateTargetTodo)
      this.clearInput()
      this.updateTargetTodo = null
    },
    async deleteTodo (todo) {
      await index.deleteObject(todo.objectID)
      this.todoList = this.todoList.filter(e => e.objectID !== objectID)
    },
    async makeTodoDone (todo) {
      todo.done = true
      await index.partialUpdateObject({
        objectID: todo.objectID,
        done: true
      })
    },
    async searchTodo () {
      let searchResult = await index.search({ query: this.query })
      this.todoList = searchResult.hits
    }
  },
  computed: {
    highlightedTodoList () {
      return this.todoList.map(todo => {
        let highlightedTodo = Object.assign({}, todo)
        if (todo._highlightResult.title.matchLevel !== 'none') {
          highlightedTodo.title = todo._highlightResult.title.value
        }
        if (todo._highlightResult.description.matchLevel !== 'none') {
          highlightedTodo.description = todo._highlightResult.description.value
        }
        return highlightedTodo
      })
    }
  }
}
</script>

<style>
.search-highlight {
  background-color: yellow;
}
</style>
