<template>
  <div class="flex justify-center items-center h-screen">
    <!-- Modal Overlay -->
    <div id="modal-overlay" class="fixed inset-0 bg-black bg-opacity-50 z-50 flex items-center justify-center hidden">
      <div class="bg-white p-8 rounded-lg shadow-md">
        <p class="text-lg font-semibold mb-4">Should this category be deleted?</p>
        <p class="text-sm text-gray-700">This category '{{ categoryToDelete }}' and all tasks within this category will
          be deleted.</p>
        <div class="flex justify-end mt-4">
          <button @click="deleteConfirmed"
            class="bg-red-500 hover:bg-red-600 text-white font-bold py-2 px-4 rounded mr-2 focus:outline-none">Delete</button>
          <button @click="closeModal"
            class="bg-gray-300 hover:bg-gray-400 text-gray-700 font-bold py-2 px-4 rounded focus:outline-none">Cancel</button>
        </div>
      </div>
    </div>

    <div class="border border-gray-300 p-4 rounded w-3/5">
      <!-- Dropdown Menu -->
      <div class="relative mb-4">
        <button @click="toggleDropdown()"
          class="border border-gray-300 rounded-md px-4 py-2 text-sm font-medium text-gray-700 bg-white flex items-center justify-between w-full focus:outline-none">
          {{ selectedCategory || 'Select a category' }}
          <svg class="h-5 w-5" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7"></path>
          </svg>
        </button>
        <!-- Dropdown Content -->
        <div v-if="isDropdownOpen" class="absolute mt-2 w-full rounded-md bg-white shadow-lg z-10">
          <ul role="list" class="py-1">
            <!-- Any Category Option -->
            <li>
              <button @click="selectCategory('Any'); closeDropdown()"
                class="text-gray-700 block px-4 py-2 text-sm hover:bg-gray-100 w-full text-left focus:outline-none">
                Any
              </button>
            </li>
            <!-- Existing Categories -->
            <li v-for="(category, index) in categories" :key="index">
              <div class="flex justify-between items-center">
                <button @click="selectCategory(category); closeDropdown()"
                  class="text-gray-700 block px-4 py-2 text-sm hover:bg-gray-100 w-full text-left focus:outline-none">
                  {{ category }}
                </button>
                <button @click="deleteCategory(category)"
                  class="text-gray-500 hover:text-red-500 focus:outline-none ml-2">
                  <svg class="h-4 w-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"
                    xmlns="http://www.w3.org/2000/svg">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12">
                    </path>
                  </svg>
                </button>
              </div>
            </li>
            <!-- New Category Input -->
            <li>
              <input v-model="newCategory" @keyup.enter="addNewCategory"
                class="border border-gray-300 rounded-md px-4 py-2 text-sm hover:bg-gray-100 w-full text-left focus:outline-none"
                type="text" placeholder="Add New Category">
            </li>
          </ul>
        </div>
      </div>

      <!-- Task Input -->
      <input class="border border-gray-300 p-2 mb-4 w-full" type="text" id="task_input" placeholder="Buy three egg."
        @keyup.enter="addTask" v-model="newTask">

      <!-- Task Statistics -->
      <div class="flex justify-between mb-4">
        <span v-if="tasks.length > 0"
          class="inline-flex items-center rounded-md bg-blue-50 px-3 py-1 text-sm font-medium text-blue-700 ring-1 ring-inset ring-blue-700/10">
          Total <span class="rounded-full bg-blue-700 text-white ml-2 px-2 pt-1">{{ tasks.length }}</span>
        </span>
        <span v-if="completed.length > 0"
          class="inline-flex items-center rounded-md bg-green-50 px-3 py-1 text-sm font-medium text-green-700 ring-1 ring-inset ring-green-600/20">
          Completed <span class="rounded-full bg-green-700 text-white ml-2 px-2 pt-1">{{ completed.length }}</span>
        </span>
        <span v-if="uncompleted.length > 0"
          class="inline-flex items-center rounded-md bg-yellow-50 px-3 py-1 text-sm font-medium text-yellow-800 ring-1 ring-inset ring-yellow-600/20">
          Not Completed <span class="rounded-full bg-yellow-700 text-white ml-2 px-2 pt-1">{{ uncompleted.length
            }}</span>
        </span>
      </div>

      <!-- Task List -->
      <div class="max-h-80 overflow-y-auto">
        <ul role="list" class="divide-y divide-gray-100">
          <li v-if="!filteredTasks.length" class="text-center py-4 text-gray-500">
            No tasks available. Try something new!
          </li>
          <li v-else v-for="task in filteredTasks" :key="task.queue"
            class="flex justify-between items-center gap-x-6 py-5">
            <div class="flex items-center gap-4">
              <input type="checkbox" @click="markTask(task.queue)"
                class="task_check appearance-none h-6 w-6 border border-gray-300 rounded-md checked:bg-blue-600 checked:border-transparent focus:outline-none">
              <div>
                <p class="text-md font-semibold leading-6 text-gray-900">{{ task.message }}</p>
                <p class="text-sm font-semibold leading-6 text-gray-500">{{ task.isCompleted ?
                  "Completed" : "Not Completed" }}</p>
              </div>
            </div>
            <button @click="deleteTask(task.queue)" class="text-red-500 hover:text-red-700 focus:outline-none">
              <svg class="h-5 w-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"
                xmlns="http://www.w3.org/2000/svg">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"></path>
              </svg>
            </button>
          </li>
        </ul>
      </div>

    </div>
  </div>
</template>

<script setup lang="ts">
import { Ref, ref, onMounted, watch, computed } from 'vue';

let index = 0;
const newTask = ref('');
const selectedCategory = ref('Any');
const isDropdownOpen = ref(false);
const newCategory = ref('');
const categoryToDelete = ref('');

type List = { message: string, isCompleted: boolean, queue: number, category: string }[];

const tasks: Ref<List> = ref([]);
const completed: Ref<List> = ref([]);
const uncompleted: Ref<List> = ref([]);

const categories = JSON.parse(localStorage.getItem('categories') ?? '["Work", "Personal", "Study", "Home"]');

onMounted(() => {
  const savedTasks = JSON.parse(localStorage.getItem('tasks') ?? '[]');
  tasks.value = savedTasks;
  completed.value = savedTasks.filter((t: any) => t.isCompleted);
  uncompleted.value = savedTasks.filter((t: any) => !t.isCompleted);
  index = savedTasks.length > 0 ? savedTasks[savedTasks.length - 1].queue + 1 : 0;
});

watch(tasks, (newTasks) => localStorage.setItem('tasks', JSON.stringify(newTasks)), { deep: true });

const addTask = () => {
  let value = newTask.value.trim();

  if (value.length < 1) return;

  let queue = index++;

  const _task = { message: value.trim(), isCompleted: false, queue, category: selectedCategory.value };

  tasks.value.unshift(_task);
  uncompleted.value.unshift(_task);

  newTask.value = '';
}

const markTask = (queue: number) => {
  const task = tasks.value.find((t) => t.queue === queue);
  if (!task) return;

  task.isCompleted = !task.isCompleted;

  completed.value = tasks.value.filter((t) => t.isCompleted);
  uncompleted.value = tasks.value.filter((t) => !t.isCompleted);
}

const deleteTask = (queue: number) => {
  tasks.value = tasks.value.filter((t) => t.queue !== queue);
  completed.value = tasks.value.filter((t) => t.isCompleted);
  uncompleted.value = tasks.value.filter((t) => !t.isCompleted);
}

const toggleDropdown = () => {
  isDropdownOpen.value = !isDropdownOpen.value;
}

const closeDropdown = () => {
  isDropdownOpen.value = false;
}

const deleteCategory = (del: string) => {
  categoryToDelete.value = del;
  (document.getElementById('modal-overlay') as any).classList.remove('hidden');
}

const deleteConfirmed = () => {
  const categoryToDeleteValue = categoryToDelete.value;
  const categoryIndex = categories.findIndex((category: any) => category === categoryToDeleteValue);
  if (categoryIndex === -1) return;

  categories.splice(categoryIndex, 1);
  localStorage.setItem('categories', JSON.stringify(categories));

  tasks.value = tasks.value.filter((task) => task.category !== categoryToDeleteValue);
  completed.value = tasks.value.filter((t) => t.isCompleted);
  uncompleted.value = tasks.value.filter((t) => !t.isCompleted);

  if (selectedCategory.value === categoryToDeleteValue) selectedCategory.value = 'Any';

  closeModal();
}

function closeModal() {
  (document.getElementById('modal-overlay') as any).classList.add('hidden');
}

const selectCategory = (category: string) => {
  selectedCategory.value = category;
  isDropdownOpen.value = false;
}

const addNewCategory = () => {
  const category = newCategory.value.trim();
  if (category.length < 1) return;

  categories.push(category);
  localStorage.setItem('categories', JSON.stringify(categories));
  selectedCategory.value = category;
  newCategory.value = '';
  isDropdownOpen.value = false;
}

const filteredTasks = computed(() => {
  if (selectedCategory.value === 'Any') return tasks.value;
  else return tasks.value.filter((task) => task.category === selectedCategory.value);
});
</script>

<style scoped>
.task_check {
  @apply appearance-none h-6 w-6 border border-gray-300 rounded-md focus:outline-none transition duration-200;
}

.task_check:checked {
  @apply bg-blue-600 border-transparent;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 20 20' fill='white'%3E%3Cpath fill-rule='evenodd' d='M16.707 5.293a1 1 0 00-1.414 0L8 12.586 4.707 9.293a1 1 0 00-1.414 1.414l4 4a1 1 0 001.414 0l8-8a1 1 0 000-1.414z' clip-rule='evenodd' /%3E%3C/svg%3E");
  background-size: 1rem;
  background-position: center;
  background-repeat: no-repeat;
}
</style>