<!-- eslint-disable vue/no-export-in-script-setup -->
<!-- eslint-disable no-unused-vars -->
<script setup>
import { onBeforeMount, ref, toRaw } from 'vue';
import { PlusIcon as PlusIconMini } from '@heroicons/vue/solid';
import { CheckCircleIcon, TrashIcon } from '@heroicons/vue/outline';
import { Web5 } from '@web5/api';

let web5;
let myDid;
const todos = ref([]);


onBeforeMount(async () => {
  // Initialize Web5
  ({ web5, did: myDid} = await Web5.connect());

  // Populate ToDos from DWN
  const { records } = await web5.dwn.records.query({
    message: {
      filter: {
        schema: 'https://some-schema-registry.org/todo'
      },
      dataSort: 'createdAscending'
    }
  });

  // Add entry to ToDos array
  for (let record of records) {
    const data = await record.data.json();
    const todo = {record, data, id: record.id};
    todos.value.push(todo);
  }

});

// Adding ToDos
const newTodoDescription = ref('');

async function addTodo() {
  const todoData = {
    completed   : false,
    description : newTodoDescription.value
  };

  newTodoDescription.value = '';

  // Create a new record
  const { record } = await web5.dwn.records.create({
    data    : todoData,
    message : {
      schema     : 'https://some-schema-registry.org/todo',
      dataFormat : 'application/json'
    }
  });

  // add DWeb message recordID as a way to reference the message for further updates
  const data = await record.data.json();
  const todo = { record, data, id: record.id };
  todos.value.push(todo);


}

// Delete ToDo
async function deleteTodo(todoItem) {
  let deleteTodo;
  let index = 0;

  for (let todo of todos.value) {
    if (todoItem.id === todo.id) {
      deleteTodo = todo;
      break;
    }
    index ++;
  }
  todos.value.splice(index, 1);

  // Delete the record
  await web5.dwn.records.delete({
    message: {
      recordId: deleteTodo.id
    }
  });

}

// Toggling ToDo Status
async function toggleTodoComplete(todoItem) {
  let toggledTodo;
  let updatedTodoData;

  for (let todo of todos.value) {
    if (todoItem.id === todo.id) {
      toggledTodo = todo;
      todo.data.completed = !todo.data.completed;
      updatedTodoData = { ...toRaw(todo.data) };
      break;
    }
  }

  // Get record in DWN
  const { record } = await web5.dwn.records.get({
    message: {
      recordId: toggledTodo.id
    }
  });

  // Update record in DWN
  await record.update({
    data: updatedTodoData
  });

}

</script>

<template>
  <div class="flex flex-col items-center justify-center min-h-full px-8 py-12 sm:px-6">
    <!-- Title -->
    <div class="sm:max-w-md sm:w-full">
      <h2 class="font-bold text-3xl text-center tracking-tight">
        Todo List
      </h2>
    </div>
      
    <!-- Add ToDos Form -->
    <div class="mt-16">
      <form class="flex space-x-4" @submit.prevent="addTodo">
        <div class="border-b border-gray-200 sm:w-full">
          <label for="add-todo" class="sr-only">Add a todo</label>
          <textarea
            rows="1" name="add-todo" id="add-todo" v-model="newTodoDescription"
            @keydown.enter.exact.prevent="addTodo"
            class="block border-0 border-transparent focus:ring-0 p-0 pb-2 resize-none sm:text-sm w-96"
            placeholder="Add a Todo" />
        </div>
        <button type="submit" class="bg-indigo-600 border border-transparent focus:outline-none focus:ring-2 focus:ring-indigo-500 focus:ring-offset-2 hover:bg-indigo-700 inline-flex items-center p-1 rounded-full shadow-sm text-white">
          <PlusIconMini class="h-5 w-5" aria-hidden="true" />
        </button>
      </form>
    </div>

    <!-- ToDos -->
    <div v-if="(todos.length > 0)" class="border-gray-200 border-t border-x mt-16 rounded-lg shadow-md sm:max-w-xl sm:mx-auto sm:w-full">
      <div v-for="todo in todos" :key="todo" class="border-b border-gray-200 flex items-center p-4">
        <div @click="toggleTodoComplete(todo)" class="cursor-pointer">
          <CheckCircleIcon class="h-8 text-gray-200 w-8" :class="{ 'text-green-500': todo.data.completed }" />
        </div>
        <div class="font-light ml-3 text-gray-500 text-xl">
          {{ todo.data.description }}
        </div>

        <!-- Delete ToDo goes here later -->
        <div class="ml-auto">
          <div @click="deleteTodo(todo)" class="cursor-pointer">
            <TrashIcon class="h-8 text-gray-200 w-8" :class="'text-red-500'" />
          </div>
        </div>
      </div>
    </div>
  </div>
</template>