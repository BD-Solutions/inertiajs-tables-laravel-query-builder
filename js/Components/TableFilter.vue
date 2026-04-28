<template>
  <ButtonWithDropdown
    placement="bottom-end"
    dusk="filters-dropdown"
    :active="hasEnabledFilters"
  >
    <template #button>
      <svg
        xmlns="http://www.w3.org/2000/svg"
        class="h-5 w-5"
        :class="{
          'text-gray-400': !hasEnabledFilters,
          'text-green-400': hasEnabledFilters,
        }"
        viewBox="0 0 20 20"
        fill="currentColor"
      >
        <path
          fill-rule="evenodd"
          d="M3 3a1 1 0 011-1h12a1 1 0 011 1v3a1 1 0 01-.293.707L12 11.414V15a1 1 0 01-.293.707l-2 2A1 1 0 018 17v-5.586L3.293 6.707A1 1 0 013 6V3z"
          clip-rule="evenodd"
        />
      </svg>
    </template>

    <div
      role="menu"
      aria-orientation="horizontal"
      aria-labelledby="filter-menu"
      class="max-h-[min(32rem,calc(100vh-7rem))] w-[22rem] max-w-[calc(100vw-2rem)] overflow-y-auto overscroll-contain p-2"
    >
      <div
        v-for="(filter, key) in filters"
        :key="key"
        class="mb-2 overflow-hidden rounded-lg border border-gray-100 bg-white last:mb-0"
      >
        <h3 class="flex items-center justify-between bg-gray-50 px-3 py-2 text-xs font-semibold uppercase tracking-wide text-gray-600">
          <span>{{ filter.label }}</span>
          <span
            v-if="isFilterActive(filter)"
            class="ml-2 rounded-full bg-green-100 px-2 py-0.5 text-[10px] font-semibold text-green-700"
          >
            Attivo
          </span>
        </h3>

        <div
          v-if="filter.type === 'select'"
          class="space-y-2 p-3"
        >
          <input
            v-if="optionCount(filter) > 7"
            type="text"
            :placeholder="`Cerca ${filter.label}...`"
            :value="searchTerms[filter.key] || ''"
            class="block w-full rounded-md border-gray-300 text-sm shadow-sm focus:border-indigo-500 focus:ring-indigo-500"
            @input="searchTerms[filter.key] = $event.target.value"
          >
          <select
            :name="filter.key"
            :value="filter.value"
            class="block w-full rounded-md border-gray-300 text-sm shadow-sm focus:border-indigo-500 focus:ring-indigo-500"
            @change="onFilterChange(filter.key, $event.target.value || null)"
          >
            <option
              v-for="(label, val) in filteredOptions(filter)"
              :key="val"
              :value="val"
            >
              {{ label }}
            </option>
          </select>
        </div>

        <div
          v-else-if="filter.type === 'multiselect'"
          class="space-y-2 p-3"
        >
          <input
            type="text"
            :placeholder="`Cerca ${filter.label}...`"
            :value="searchTerms[filter.key] || ''"
            class="block w-full rounded-md border-gray-300 text-sm shadow-sm focus:border-indigo-500 focus:ring-indigo-500"
            @input="searchTerms[filter.key] = $event.target.value"
          >
          <div
            v-if="filteredOptionCount(filter) > 0"
            class="max-h-40 space-y-1 overflow-y-auto pr-1"
          >
            <label
              v-for="(label, val) in filteredOptions(filter)"
              :key="val"
              class="flex cursor-pointer items-center gap-2 rounded-md px-2 py-1.5 text-sm text-gray-700 hover:bg-gray-50"
            >
              <input
                type="checkbox"
                :value="val"
                :checked="isMultiSelected(filter, val)"
                class="rounded border-gray-300 text-indigo-600 focus:ring-indigo-500"
                @change="toggleMultiSelect(filter.key, filter.value, val, $event.target.checked)"
              >
              {{ label }}
            </label>
          </div>
          <p
            v-else
            class="rounded-md bg-gray-50 px-3 py-2 text-sm text-gray-500"
          >
            {{ optionCount(filter) > 0 ? 'Nessuna opzione trovata.' : 'Nessuna opzione disponibile.' }}
          </p>
          <div
            v-if="Array.isArray(filter.value) && filter.value.length > 0"
            class="flex items-center justify-between border-t border-gray-100 pt-2"
          >
            <span class="text-xs text-gray-500">
              {{ filter.value.length }} selezionati
            </span>
            <button
              type="button"
              class="text-xs font-medium text-indigo-600 hover:text-indigo-800"
              @click="onFilterChange(filter.key, null)"
            >
              Cancella
            </button>
          </div>
        </div>

        <div
          v-else-if="filter.type === 'boolean'"
          class="p-3"
        >
          <div class="grid grid-cols-3 overflow-hidden rounded-md border border-gray-300 bg-white shadow-sm">
            <button
              v-for="opt in booleanOptions"
              :key="String(opt.val)"
              type="button"
              class="border-r border-gray-300 px-3 py-2 text-sm font-medium transition-colors last:border-r-0"
              :class="filter.value === opt.val
                ? 'bg-indigo-600 text-white'
                : 'bg-white text-gray-700 hover:bg-gray-50'"
              @click="onFilterChange(filter.key, opt.val)"
            >
              {{ opt.label }}
            </button>
          </div>
        </div>

        <div
          v-else-if="filter.type === 'date'"
          class="space-y-2 p-3"
        >
          <input
            type="date"
            :value="filter.value || ''"
            class="block w-full rounded-md border-gray-300 text-sm shadow-sm focus:border-indigo-500 focus:ring-indigo-500"
            @change="onFilterChange(filter.key, $event.target.value || null)"
          >
          <button
            v-if="filter.value"
            type="button"
            class="text-xs font-medium text-indigo-600 hover:text-indigo-800"
            @click="onFilterChange(filter.key, null)"
          >
            Clear date
          </button>
        </div>

        <div
          v-else-if="filter.type === 'date-range'"
          class="space-y-3 p-3"
        >
          <div class="grid grid-cols-1 gap-3 sm:grid-cols-2">
            <div>
              <label class="mb-1 block text-xs font-medium text-gray-500">Da</label>
              <input
                type="date"
                :value="filter.value && filter.value.from ? filter.value.from : ''"
                class="block w-full rounded-md border-gray-300 text-sm shadow-sm focus:border-indigo-500 focus:ring-indigo-500"
                @change="onDateRangeChange(filter.key, filter.value, 'from', $event.target.value)"
              >
            </div>
            <div>
              <label class="mb-1 block text-xs font-medium text-gray-500">A</label>
              <input
                type="date"
                :value="filter.value && filter.value.to ? filter.value.to : ''"
                class="block w-full rounded-md border-gray-300 text-sm shadow-sm focus:border-indigo-500 focus:ring-indigo-500"
                @change="onDateRangeChange(filter.key, filter.value, 'to', $event.target.value)"
              >
            </div>
          </div>
          <button
            v-if="isFilterActive(filter)"
            type="button"
            class="text-xs font-medium text-indigo-600 hover:text-indigo-800"
            @click="onFilterChange(filter.key, null)"
          >
            Cancella date
          </button>
        </div>
      </div>
    </div>
  </ButtonWithDropdown>
</template>

<script setup>
import { ref } from "vue";
import ButtonWithDropdown from "./ButtonWithDropdown.vue";

const props = defineProps({
    hasEnabledFilters: {
        type: Boolean,
        required: true,
    },

    filters: {
        type: Object,
        required: true,
    },

    onFilterChange: {
        type: Function,
        required: true,
    },
});

const searchTerms = ref({});

const booleanOptions = [
    { val: null, label: "Tutti" },
    { val: "1", label: "Si" },
    { val: "0", label: "No" },
];

function optionCount(filter) {
    return Object.keys(filter.options || {}).length;
}

function filteredOptions(filter) {
    const term = (searchTerms.value[filter.key] || "").toLowerCase();
    const options = filter.options || {};
    if (!term) return options;
    return Object.fromEntries(
        Object.entries(options).filter(([, label]) =>
            String(label).toLowerCase().includes(term)
        )
    );
}

function filteredOptionCount(filter) {
    return Object.keys(filteredOptions(filter) || {}).length;
}

function isMultiSelected(filter, val) {
    return Array.isArray(filter.value) && filter.value.map(String).includes(String(val));
}

function isFilterActive(filter) {
    const v = filter.value;
    if (v === null || v === undefined || v === "") return false;
    if (Array.isArray(v)) return v.length > 0;
    if (typeof v === "object") return Object.values(v).some(Boolean);
    return true;
}

function toggleMultiSelect(key, currentValue, val, checked) {
    const current = Array.isArray(currentValue) ? currentValue.map(String) : [];
    const strVal = String(val);

    const updated = checked
        ? [...new Set([...current, strVal])]
        : current.filter((v) => v !== strVal);

    props.onFilterChange(key, updated.length > 0 ? updated : null);
}

function onDateRangeChange(key, currentValue, field, rawValue) {
    const updated = {
        from: currentValue?.from || null,
        to: currentValue?.to || null,
        [field]: rawValue || null,
    };
    const hasAny = updated.from || updated.to;
    props.onFilterChange(key, hasAny ? updated : null);
}
</script>
