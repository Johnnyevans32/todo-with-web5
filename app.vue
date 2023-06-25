<template>
  <div
    class="flex flex-col gap-4 items-center text-white p-10 bg-black min-h-screen"
  >
    <div class="flex justify-between min-w-full">
      <h1>track productivity with web5</h1>
      <div>{{ formattedDid() }}</div>
    </div>

    <div>
      <label for="add-todo" class="sr-only">Add a todo</label>
      <input
        type="text"
        id="add-todo"
        v-model="todoDescription"
        @keydown.enter.exact.prevent="addTodo"
        class="p-10 w-72 bg-inherit border-2 border-white"
        placeholder="what are you working on?"
      />
    </div>
    <button type="submit" :class="btnClass" class="w-72" @click="addTodo">
      add
    </button>
    <div v-if="!Object.values(todos).length">
      <h1 class="text-red-600 text-3xl">
        you currently have nothing been tracked
      </h1>
    </div>
    <div
      v-else
      class="border-2 border-white bg-transparent grid grid-cols-4 gap-2 p-4 min-w-full text-xs sm:text-xl"
    >
      <p>completed?</p>
      <p>task name</p>
      <p>time started</p>
      <p></p>
    </div>
    <div
      v-for="todo in todos"
      :key="todo.id"
      :class="todo.data.completed && 'line-through'"
      class="border-2 border-white bg-transparent grid grid-cols-4 gap-2 p-4 min-w-full text-xs items-center sm:text-xl"
    >
      <input
        type="checkbox"
        class="accent-green-500 cursor-pointer w-10 h-5"
        @click="toggleTodoStatus(todo.id)"
        :checked="todo.data.completed"
      />
      <p>
        {{ todo.data.description }}
      </p>
      <p>
        {{ formatDate(todo.data.createdAt) }}
      </p>
      <button
        @click="deleteTodo(todo.id)"
        :class="btnClass"
        class="sm:w-36 sm:h-10 h-10 w-15 justify-self-end"
      >
        delete
      </button>
    </div>

    <div class="flex justify-between min-w-full">
      <p>&copy; 0xjevan</p>
      <p>powered by web5</p>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent } from "vue";
import { Web5 } from "@tbd54566975/web5";
import { DateSort } from "@tbd54566975/dwn-sdk-js";
import moment from "moment";
import { Record } from "@tbd54566975/web5/dist/types/record";
import { ref, onBeforeMount } from "./.nuxt/imports";

interface ITodoData {
  completed: boolean;
  description: string;
  createdAt: Date;
}

interface ITodo {
  record: Record;
  data: ITodoData;
  id: string;
}

export default defineComponent({
  async setup() {
    let web5: Web5;
    let did: string;
    const myDid = ref<string>("");
    const todoDescription = ref<string>("");
    const todos = ref<{ [recordId: string]: ITodo }>({});

    const btnClass = ref(
      "bg-gradient-to-r from-green-400 to-blue-500 hover:from-pink-500 hover:to-yellow-500  inline-flex items-center justify-center p-1 shadow-sm"
    );
    onBeforeMount(async () => {
      try {
        ({ web5, did } = await Web5.connect());
        myDid.value = did;

        const { records } = await web5.dwn.records.query({
          message: {
            filter: {
              schema: "http://some-schema-registry.org/todo",
            },
            dateSort: DateSort.CreatedAscending,
          },
        });

        const loadRecords = (records || []).map(async (record) => {
          const data = (await record.data.json()) as ITodoData;

          todos.value[record.id] = { record, data, id: record.id } as ITodo;
        });

        await Promise.all(loadRecords);
      } catch (err) {
        console.log("before mount error", { err });
      }
    });

    const formattedDid = () =>
      `${myDid.value.substring(0, 7)}...${myDid.value.slice(
        myDid.value.length - 4
      )}`;

    const formatDate = (date: Date) =>
      moment(date).format("dddd, Do MMM YYYY, h:mm:ss A");

    const failIfNull = (value: any, err: string) => {
      if (!value) {
        throw new Error(err);
      }
    };
    async function addTodo() {
      try {
        failIfNull(web5, "web5 not intialised");

        if (!todoDescription.value) {
          return;
        }
        const data = {
          completed: false,
          description: todoDescription.value,
          createdAt: new Date(),
        };

        todoDescription.value = "";

        const writeRes = await web5.dwn.records.create({
          data,
          message: {
            schema: "http://some-schema-registry.org/todo",
            dataFormat: "application/json",
          },
        });

        const record = writeRes.record as Record;

        failIfNull(record, "record not intialised");

        todos.value[record.id] = { record, data, id: record.id };
      } catch (err) {
        console.log("add todo err", { err });
      }
    }

    async function deleteTodo(recordId: string) {
      try {
        failIfNull(web5, "web5 not intialised");

        delete todos.value[recordId];

        // Delete the record in DWN
        await web5.dwn.records.delete({
          message: {
            recordId,
          },
        });
      } catch (err) {
        console.log("delete todo err", { err });
      }
    }

    // Toggling ToDo Status
    async function toggleTodoStatus(recordId: string) {
      try {
        failIfNull(web5, "web5 not intialised");
        failIfNull(todos.value[recordId], "record not found");

        todos.value[recordId].data.completed =
          !todos.value[recordId].data.completed;

        // Get record in DWN
        const { record } = await web5.dwn.records.read({
          message: {
            recordId,
          },
        });

        // Update the record in DWN
        await record.update({ data: todos.value[recordId].data });
      } catch (err) {
        console.log("update todo err", { err });
      }
    }

    return {
      myDid,
      todos,
      btnClass,
      todoDescription,
      addTodo,
      deleteTodo,
      toggleTodoStatus,
      formatDate,
      formattedDid,
    };
  },
});
</script>

<style>
html {
  font-family: "Space Grotesk", sans-serif;
}

*,
*::before,
*::after {
  box-sizing: border-box;
  margin: 0;
}
</style>
