<script setup lang="ts">
import { mapApiError } from '@/api/apiErrors';
import { useToast } from '@/common/composables/useToast';
import TaskForm from '@/modules/tasks/components/TaskForm.vue';
import type { CreateTaskDto } from '@/modules/tasks/models/create-task.dto';
import type { Task, TaskPriority } from '@/modules/tasks/models/task.model';
import {
  completeAllPendingTasks,
  completeTask,
  createTask,
  deleteAllCompletedTasks,
  getTasks,
} from '@/modules/tasks/services/tasks.service';
import { onMounted, ref, computed } from 'vue';
import { RouterLink } from 'vue-router';

const toast = useToast();

const tasks = ref<Task[]>([]);
const loading = ref(false);
const error = ref<string | null>(null);

const search = ref('');

type SortField = 'date' | 'priority' | null;
type SortDirection = 'asc' | 'desc';

const sortField = ref<SortField>(null);
const sortDirection = ref<SortDirection>('asc');

type ViewMode = 'list' | 'cards';
const viewMode = ref<ViewMode>('list');

const priorityValue = (p: TaskPriority) => {
  if (p === 'high') return 3;
  if (p === 'medium') return 2;
  return 1;
};

const isActiveSort = (field: SortField) => sortField.value === field;

const toggleSort = (field: SortField) => {
  if (sortField.value === field) {
    sortDirection.value = sortDirection.value === 'asc' ? 'desc' : 'asc';
  } else {
    sortField.value = field;
    sortDirection.value = 'asc';
  }
};

const clearSort = () => {
  sortField.value = null;
  sortDirection.value = 'asc';
};

const toggleView = () => {
  viewMode.value = viewMode.value === 'list' ? 'cards' : 'list';
};

async function loadTasks() {
  loading.value = true;
  error.value = null;

  try {
    tasks.value = await getTasks();
  } catch {
    error.value = 'No se han podido cargar las tareas.';
  } finally {
    loading.value = false;
  }
}

const filteredTasks = computed(() => {
  const q = search.value.trim().toLowerCase();

  let result = tasks.value.filter(
    (t) => t.name.toLowerCase().includes(q) || (t.description?.toLowerCase().includes(q) ?? false),
  );

  if (!sortField.value) return result;

  result = [...result].sort((a, b) => {
    let valueA: number;
    let valueB: number;

    if (sortField.value === 'date') {
      valueA = new Date(a.dueDate).getTime();
      valueB = new Date(b.dueDate).getTime();
    } else {
      valueA = priorityValue(a.priority);
      valueB = priorityValue(b.priority);
    }

    return sortDirection.value === 'asc' ? valueA - valueB : valueB - valueA;
  });

  return result;
});

async function onCreateTask(payload: CreateTaskDto) {
  try {
    await createTask(payload);
    toast.success('Tarea creada correctamente');
    await loadTasks();
  } catch (e) {
    toast.error(mapApiError(e, 'No se ha podido crear la tarea'));
  }
}

async function onCompleteTask(taskId: string) {
  try {
    await completeTask(taskId);
    toast.success('Tarea completada');
    await loadTasks();
  } catch (e) {
    toast.error(mapApiError(e, 'No se ha podido completar la tarea'));
  }
}

async function onCompleteAll() {
  try {
    await completeAllPendingTasks();
    toast.success('Todas las tareas pendientes se han completado');
    await loadTasks();
  } catch (e) {
    toast.error(mapApiError(e, 'No se han podido completar las tareas'));
  }
}

async function onDeleteCompleted() {
  try {
    await deleteAllCompletedTasks();
    toast.success('Tareas completadas eliminadas');
    await loadTasks();
  } catch (e) {
    toast.error(mapApiError(e, 'No se han podido eliminar las tareas completadas'));
  }
}

function priorityBadge(p: TaskPriority) {
  if (p === 'high') return 'bg-danger';
  if (p === 'medium') return 'bg-primary';
  return 'bg-secondary';
}

onMounted(loadTasks);
</script>

<template>
  <section class="container py-4">
    <header class="d-flex align-items-center justify-content-between mb-3">
      <h1 class="h4 mb-0">Mis tareas</h1>
      <RouterLink to="/" class="btn btn-outline-secondary btn-sm">Volver a Home</RouterLink>
    </header>

    <TaskForm :disabled="loading" @submit="onCreateTask" />

    <div class="mb-3">
      <p>Filtro de búsqueda:</p>
      <input
        v-model="search"
        type="text"
        class="form-control"
        placeholder="Buscar por nombre o descripción..."
      />
    </div>

    <div class="d-flex align-items-center gap-2 mb-3 flex-wrap border-bottom pb-2">
      <div class="d-flex gap-2">
        <button
          class="btn btn-sm"
          :class="isActiveSort('date') ? 'btn-primary' : 'btn-outline-primary'"
          @click="toggleSort('date')"
        >
          Fecha
          <span v-if="sortField === 'date'">
            {{ sortDirection === 'asc' ? '↑' : '↓' }}
          </span>
        </button>

        <button
          class="btn btn-sm"
          :class="isActiveSort('priority') ? 'btn-primary' : 'btn-outline-primary'"
          @click="toggleSort('priority')"
        >
          Prioridad
          <span v-if="sortField === 'priority'">
            {{ sortDirection === 'asc' ? '↑' : '↓' }}
          </span>
        </button>

        <button v-if="sortField" class="btn btn-sm btn-outline-secondary" @click="clearSort">
          Limpiar orden
        </button>
      </div>

      <div class="d-flex gap-2 ms-auto">
        <button class="btn btn-sm btn-outline-dark" @click="toggleView">
          {{ viewMode === 'list' ? 'Vista tarjetas' : 'Vista lista' }}
        </button>

        <button class="btn btn-sm btn-outline-secondary" :disabled="loading" @click="loadTasks">
          Recargar
        </button>
      </div>
    </div>
    <div class="d-flex align-items-center gap-2 mb-3 flex-wrap">
      <div class="d-flex gap-2">
        <button
          class="btn btn-sm btn-outline-success"
          :disabled="tasks.length === 0 || loading"
          @click="onCompleteAll"
        >
          Completar todas
        </button>

        <button
          class="btn btn-sm btn-outline-danger"
          :disabled="!tasks.some((t) => t.completed) || loading"
          @click="onDeleteCompleted"
        >
          Eliminar completadas
        </button>
      </div>
    </div>

    <div v-if="loading" class="text-center py-4">
      <div class="spinner-border" role="status"></div>
    </div>

    <div v-if="error" class="alert alert-danger">
      {{ error }}
    </div>

    <div v-if="!loading && viewMode === 'list' && filteredTasks.length" class="list-group">
      <RouterLink
        v-for="task in filteredTasks"
        :key="task.id"
        :to="{ name: 'task-detail', params: { id: task.id } }"
        class="list-group-item list-group-item-action d-flex justify-content-between align-items-center"
      >
        <div class="me-3">
          <span class="badge me-2" :class="task.completed ? 'bg-success' : 'bg-warning text-dark'">
            {{ task.completed ? 'Hecha' : 'Pendiente' }}
          </span>

          <span class="fw-semibold">{{ task.name }}</span>

          <span class="badge ms-2" :class="priorityBadge(task.priority)">
            {{ task.priority }}
          </span>

          <div class="text-muted small mt-1">
            {{ task.description }}
          </div>
        </div>

        <div class="d-flex align-items-center gap-2">
          <small class="text-muted">
            {{ new Date(task.dueDate).toLocaleDateString() }}
          </small>

          <button
            v-if="!task.completed"
            class="btn btn-sm btn-outline-success"
            @click.prevent="onCompleteTask(task.id)"
          >
            ✓
          </button>
        </div>
      </RouterLink>
    </div>

    <div v-else-if="!loading && viewMode === 'cards' && filteredTasks.length" class="row g-3">
      <div v-for="task in filteredTasks" :key="task.id" class="col-md-4">
        <div class="card h-100">
          <div class="card-body">
            <span
              class="badge mb-2"
              :class="task.completed ? 'bg-success' : 'bg-warning text-dark'"
            >
              {{ task.completed ? 'Hecha' : 'Pendiente' }}
            </span>

            <h5 class="card-title">{{ task.name }}</h5>

            <span class="badge mb-2" :class="priorityBadge(task.priority)">
              {{ task.priority }}
            </span>

            <p class="text-muted small">{{ task.description }}</p>

            <small class="text-muted d-block mb-2">
              {{ new Date(task.dueDate).toLocaleDateString() }}
            </small>

            <button
              v-if="!task.completed"
              class="btn btn-sm btn-outline-success"
              @click.prevent="onCompleteTask(task.id)"
            >
              ✓
            </button>
          </div>
        </div>
      </div>
    </div>

    <p v-else-if="!loading && search && filteredTasks.length === 0" class="text-muted">
      No hay tareas que coincidan con la búsqueda
    </p>

    <p v-else-if="!loading && tasks.length === 0" class="text-muted">No tienes tareas todavía.</p>
  </section>
</template>
