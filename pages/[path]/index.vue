<template>
  <NuxtLayout :name="data.Data.layoutName ? data.Data.layoutName : 'default'">
    <WidgetsBreadcrumb />
    <template v-for="(contents, key) in data?.widgets" #[key]>
      <component-renderer
        :key="key"
        :components="contents"
      ></component-renderer>
    </template>

    <LazyCommonRefetchButton @click="refresh">
      {{ data && !pending ? "Fetch Newest Data" : "Fetching data..." }}
    </LazyCommonRefetchButton>

    <template #empty>
      <CommonEmpty v-if="data" :show-empty="showEmpty" :data="data" />
    </template>
  </NuxtLayout>
</template>

<script setup>
import { getContentByPageSlug, getContentById } from "~~/utils/dataFetching";
import transformContent from "~~/utils/transformContent";

const route = useRoute();
const currentPath = route.path;

const { data, refresh, pending } = await useAsyncData(
  currentPath,
  async ({ $gqlClient }) => {
    // If it is inside the iframe (has isNewPage and contentId query), fetch using contentId
    if (route.query.isNewPage === "true" && route.query.contentId) {
      const { data: newResponse } = await getContentById(
        $gqlClient,
        route.query.contentId,
        {
          deep: true,
        }
      );

      return transformContent(newResponse);
    }
    const { data: response } = await getContentByPageSlug(
      $gqlClient,
      currentPath,
      {
        deep: true,
      }
    );

    return transformContent(response);
  }
);

const showEmpty = computed(() => {
  const { Data } = data.value;
  return !Data.layoutName && !Data.placeholder && !Data.contentTitle;
});

const updateContentById = (content, id, newContent, cache = {}) => {
  if (cache[content.ContentID]) return null;
  cache[content.ContentID] = true;
  if (content.ContentID === id) {
    content.Data = newContent;
    return content;
  }
  const componentDataKeys = Object.keys(content.Data);
  for (let i = 0; i < componentDataKeys.length; i++) {
    const componentDataKey = componentDataKeys[i];
    if (Array.isArray(content.Data[componentDataKey])) {
      for (let j = 0; j < content.Data[componentDataKey].length; j++) {
        const found = updateContentById(
          content.Data[componentDataKey][j],
          id,
          newContent,
          cache
        );
        if (found) {
          content.Data[componentDataKey][j] = found;
        }
      }
    }
  }
  return content;
};

const { $nimvioSdk } = useNuxtApp();
onBeforeMount(() => {
  $nimvioSdk.livePreviewUtils.onPreviewContentChange((formData) => {
    const newContent = updateContentById(
      data.value,
      formData.id,
      formData.formData
    );
    if (newContent) {
      data.value = transformContent(newContent);
    }
  });
});

useHead({
  title: data.value?.Data.pageTitle,
});
</script>

<style>
.media-wrapper {
  padding: 20px;
}

.media-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 20px;
  box-shadow: 0 0 0 0.08rem rgba(8, 60, 130, 0.06),
    0rem 0rem 1.25rem rgba(30, 34, 40, 0.04);
  border: 0;
  border-radius: 10px;
  margin-bottom: 30px;
}

.media-title {
  margin-bottom: 10px;
  line-height: 20px;
}

.media-published {
  margin-bottom: 10px;
}

a.media-download {
  border: 2px solid #512da8;
  color: #512da8;
  padding: 10px 30px;
  transition: 0.1s;
  text-decoration: none;
}

.media-download:hover {
  background-color: #512da8;
  color: #fff !important;
  transition: all 0.3s ease-in-out;
  transform: scale(1.1);
  border: none;
}

@media screen and (min-width: 768px) {
  .media-item {
    flex-direction: row;
    justify-content: space-between;
    align-items: center;
  }

  .media-info {
    text-align: left;
    padding-right: 40px;
  }
}
</style>
