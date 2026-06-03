<script setup>
import { ref, computed, onMounted, onBeforeUnmount } from "vue";
import { useRouter } from "vue-router";
import { usePasteService } from "@/modules/paste";
import { useAuthStore } from "@/stores/authStore.js";
import { useThemeMode } from "@/composables/core/useThemeMode.js";
import { formatDateTime } from "@/utils/timeUtils.js";
import { createLogger } from "@/utils/logger.js";
import { useCreatorBadge } from "@/composables/admin-management/useCreatorBadge.js";
import PermissionManager from "@/components/common/PermissionManager.vue";
import CommonPagination from "@/components/common/CommonPagination.vue";
import PastePreviewModal from "@/modules/paste/admin/components/PastePreviewModal.vue";
import PasteEditModal from "@/modules/paste/admin/components/PasteEditModal.vue";

const log = createLogger("MyPastes");
const router = useRouter();
const authStore = useAuthStore();
const pasteService = usePasteService();
const { isDarkMode: darkMode } = useThemeMode();
const { formatCreator } = useCreatorBadge();

const hasPermission = computed(() => authStore.hasTextSharePermission);

const pastes = ref([]);
const loading = ref(false);
const searchQuery = ref("");
const isSearchMode = ref(false);
const searchLoading = ref(false);
const pagination = ref(null);
const lastRefreshTime = ref("");
const previewPaste = ref(null);
const showPreview = ref(false);
const editingPaste = ref(null);
const showEdit = ref(false);
let searchTimer = null;
let cancellationToken = { cancelled: false };

const pageSizeOptions = [10, 20, 30, 50, 100];

const formatDate = (d) => d ? formatDateTime(d) : "-";

const loadPastes = async (offset = 0) => {
  if (cancellationToken.cancelled) return;
  loading.value = true;
  try {
    const params = { limit: 10, offset };
    if (pagination.value) params.limit = pagination.value.limit;
    if (isSearchMode.value && searchQuery.value.trim()) {
      searchLoading.value = true;
      params.search = searchQuery.value.trim();
    }
    const result = await pasteService.getPastes(params);
    if (cancellationToken.cancelled) return;
    pastes.value = result.items || [];
    pagination.value = result.pagination || null;
    lastRefreshTime.value = new Date().toLocaleTimeString("zh-CN");
  } catch (e) {
    log.error("加载失败:", e);
    if (!cancellationToken.cancelled) pastes.value = [];
  } finally {
    loading.value = false;
    searchLoading.value = false;
  }
};

const refreshPastes = () => { loadPastes(0); };

const clearSearch = () => {
  searchQuery.value = "";
  isSearchMode.value = false;
  loadPastes(0);
};

const debouncedSearch = () => {
  clearTimeout(searchTimer);
  searchTimer = setTimeout(() => {
    if (searchQuery.value.trim()) {
      isSearchMode.value = true;
    } else {
      isSearchMode.value = false;
    }
    loadPastes(0);
  }, 300);
};

const handleOffsetChangeWithSearch = (offset) => { loadPastes(offset); };
const handlePageSizeChange = (limit) => {
  if (pagination.value) pagination.value.limit = limit;
  loadPastes(0);
};

const goToViewPage = (slug) => { window.open(`/paste/${slug}`, "_blank"); };

const openPreview = (paste) => {
  if (!paste) return;
  pasteService.getPasteById(paste.id).then((data) => {
    previewPaste.value = data;
    showPreview.value = true;
  }).catch((e) => { log.error("获取预览失败:", e); });
};
const closePreview = () => { showPreview.value = false; previewPaste.value = null; };

const openEdit = (paste) => { editingPaste.value = paste; showEdit.value = true; };
const closeEdit = () => { showEdit.value = false; editingPaste.value = null; };
const submitEdit = async (updated) => {
  try {
    await pasteService.updatePaste(editingPaste.value.slug, updated);
    closeEdit();
    refreshPastes();
  } catch (e) {
    log.error("编辑失败:", e);
    alert("编辑失败: " + (e.message || "未知错误"));
  }
};

const deletePaste = async (id) => {
  if (!confirm("确定删除？")) return;
  try {
    await pasteService.deleteSinglePaste(id);
    pastes.value = pastes.value.filter(p => p.id !== id);
    if (pagination.value) pagination.value.total = Math.max(0, (pagination.value.total || 1) - 1);
  } catch (e) {
    log.error("删除失败:", e);
    alert("删除失败");
  }
};

const navigateToAdmin = () => { router.push("/admin"); };

onMounted(() => { loadPastes(0); });
onBeforeUnmount(() => { cancellationToken.cancelled = true; });
</script>

<template>
  <div class="my-pastes-page mx-auto px-3 sm:px-6 flex-1 flex flex-col pt-6 sm:pt-8 w-full max-w-full sm:max-w-6xl">
    <div class="header mb-5 pb-3 border-b" :class="darkMode ? 'border-gray-700' : 'border-gray-200'">
      <h2 class="text-xl font-semibold" :class="darkMode ? 'text-gray-100' : 'text-gray-900'">我的文本</h2>
    </div>

    <PermissionManager
      :dark-mode="darkMode"
      permission-type="text"
      :permission-required-text="$t('markdown.permissionRequired')"
      :login-auth-text="$t('mount.loginAuth')"
      @navigate-to-admin="navigateToAdmin"
    />

    <div v-if="hasPermission" class="main-content flex-1 flex flex-col">
      <div class="flex items-center gap-3 mb-4">
        <div class="relative flex-1">
          <svg class="absolute left-3 top-1/2 -translate-y-1/2 w-4 h-4 text-gray-400 pointer-events-none" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z"/></svg>
          <input
            v-model="searchQuery"
            @input="debouncedSearch"
            placeholder="搜索文本..."
            class="w-full pl-10 pr-8 py-2 text-sm border rounded-lg outline-none transition-colors focus:ring-2 focus:ring-primary-500/30 focus:border-primary-500"
            :class="darkMode ? 'bg-gray-800 border-gray-600 text-gray-200 placeholder-gray-500' : 'bg-white border-gray-300 text-gray-900 placeholder-gray-400'"
          />
          <button v-if="searchQuery" @click="clearSearch" class="absolute right-2 top-1/2 -translate-y-1/2 w-5 h-5 flex items-center justify-center rounded-full text-gray-400 hover:text-gray-600 dark:hover:text-gray-300 hover:bg-gray-200 dark:hover:bg-gray-600 transition-colors">
            <svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"/></svg>
          </button>
        </div>
        <button
          class="inline-flex items-center gap-1.5 px-4 py-2 text-sm font-medium rounded-lg border transition-colors shrink-0"
          :class="darkMode ? 'border-gray-600 bg-gray-800 text-gray-300 hover:bg-gray-700' : 'border-gray-300 bg-white text-gray-700 hover:bg-gray-50'"
          @click="refreshPastes" :disabled="loading || searchLoading"
        >
          <svg class="h-4 w-4" :class="(loading || searchLoading) ? 'animate-spin' : ''" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 4v5h.582m15.356 2A8.001 8.001 0 004.582 9m0 0H9m11 11v-5h-.581m0 0a8.003 8.003 0 01-15.357-2m15.357 2H15"/></svg>
          刷新
        </button>
      </div>

      <div class="flex items-center justify-between mb-3 text-xs text-gray-500 dark:text-gray-400" v-if="lastRefreshTime">
        <span>上次刷新: {{ lastRefreshTime }}</span>
        <span v-if="pagination">共 {{ pagination.total || 0 }} 条</span>
      </div>

      <div class="border rounded-lg overflow-hidden shadow-sm" :class="darkMode ? 'border-gray-700 bg-gray-800' : 'border-gray-200 bg-white'">
        <table class="w-full table-fixed">
          <thead>
            <tr class="border-b-2" :class="darkMode ? 'border-gray-600 bg-gray-800/50' : 'border-gray-300 bg-gray-50/80'">
              <th class="text-left px-3 sm:px-4 py-3 text-xs font-semibold tracking-wider uppercase w-[30%]" :class="darkMode ? 'text-gray-400' : 'text-gray-500'">标题</th>
              <th class="text-left px-3 sm:px-4 py-3 text-xs font-semibold tracking-wider uppercase hidden sm:table-cell w-[20%]" :class="darkMode ? 'text-gray-400' : 'text-gray-500'">备注</th>
              <th class="text-left px-3 sm:px-4 py-3 text-xs font-semibold tracking-wider uppercase hidden md:table-cell w-[10%]" :class="darkMode ? 'text-gray-400' : 'text-gray-500'">创建者</th>
              <th class="text-left px-3 sm:px-4 py-3 text-xs font-semibold tracking-wider uppercase hidden sm:table-cell w-[20%]" :class="darkMode ? 'text-gray-400' : 'text-gray-500'">创建时间</th>
              <th class="text-right px-3 sm:px-4 py-3 text-xs font-semibold tracking-wider uppercase w-[20%]" :class="darkMode ? 'text-gray-400' : 'text-gray-500'">操作</th>
            </tr>
          </thead>
          <tbody class="divide-y" :class="darkMode ? 'divide-gray-700/50' : 'divide-gray-100'">
            <tr v-if="loading" class="h-40">
              <td colspan="5" class="text-center">
                <svg class="animate-spin h-6 w-6 mx-auto" :class="darkMode ? 'text-gray-500' : 'text-gray-400'" fill="none" viewBox="0 0 24 24"><circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"/><path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z"/></svg>
              </td>
            </tr>
            <tr v-else-if="pastes.length === 0" class="h-40">
              <td colspan="5" class="text-center">
                <p class="text-sm" :class="darkMode ? 'text-gray-500' : 'text-gray-400'">{{ searchQuery ? '没有找到匹配的文本' : '暂无文本' }}</p>
              </td>
            </tr>
            <tr
              v-for="doc in pastes"
              :key="doc.id"
              class="group cursor-pointer transition-colors hover:bg-primary-50/30 dark:hover:bg-primary-900/10"
              @click="goToViewPage(doc.slug)"
            >
              <td class="px-3 sm:px-4 py-3.5">
                <span class="text-sm font-medium truncate block group-hover:text-primary-600 dark:group-hover:text-primary-400 transition-colors" :class="darkMode ? 'text-gray-200' : 'text-gray-900'">
                  {{ doc.title || doc.slug || '未命名文档' }}
                </span>
              </td>
              <td class="px-3 sm:px-4 py-3.5 hidden sm:table-cell">
                <span class="text-sm truncate block" :class="darkMode ? 'text-gray-400' : 'text-gray-500'">{{ doc.remark || '-' }}</span>
              </td>
              <td class="px-3 sm:px-4 py-3.5 hidden md:table-cell">
                <span class="text-sm truncate block" :class="darkMode ? 'text-gray-400' : 'text-gray-500'">{{ formatCreator(doc) }}</span>
              </td>
              <td class="px-3 sm:px-4 py-3.5 hidden sm:table-cell">
                <span class="text-sm truncate block" :class="darkMode ? 'text-gray-400' : 'text-gray-500'">{{ formatDate(doc.created_at) }}</span>
              </td>
              <td class="px-3 sm:px-4 py-3.5 text-right" @click.stop>
                <div class="flex items-center justify-end gap-1 sm:gap-2 whitespace-nowrap">
                  <button @click="openPreview(doc)" class="px-1.5 sm:px-2.5 py-1 text-xs rounded-md border transition-colors hover:bg-gray-100 dark:hover:bg-gray-700" :class="darkMode ? 'border-gray-600 text-gray-400 hover:text-gray-200' : 'border-gray-200 text-gray-500 hover:text-gray-700'">预览</button>
                  <button @click="openEdit(doc)" class="px-1.5 sm:px-2.5 py-1 text-xs rounded-md border transition-colors hover:bg-gray-100 dark:hover:bg-gray-700" :class="darkMode ? 'border-gray-600 text-gray-400 hover:text-gray-200' : 'border-gray-200 text-gray-500 hover:text-gray-700'">编辑</button>
                  <button @click="deletePaste(doc.id)" class="px-1.5 sm:px-2.5 py-1 text-xs rounded-md border transition-colors hover:bg-red-50 dark:hover:bg-red-900/20" :class="darkMode ? 'border-red-800 text-red-400 hover:text-red-300' : 'border-red-200 text-red-500 hover:text-red-700'">删除</button>
                </div>
              </td>
            </tr>
          </tbody>
        </table>
      </div>

      <div class="mt-4 mb-4">
        <CommonPagination
          :dark-mode="darkMode"
          :pagination="pagination"
          :page-size-options="pageSizeOptions"
          :search-mode="isSearchMode"
          :search-term="searchQuery"
          mode="offset"
          @offset-changed="handleOffsetChangeWithSearch"
          @limit-changed="handlePageSizeChange"
        />
      </div>
    </div>

    <PastePreviewModal
      :dark-mode="darkMode"
      :show-preview="showPreview"
      :paste="previewPaste"
      @close="closePreview"
      @view-paste="goToViewPage"
    />

    <PasteEditModal
      :dark-mode="darkMode"
      :show-edit="showEdit"
      :paste="editingPaste"
      @close="closeEdit"
      @save="submitEdit"
    />
  </div>
</template>
