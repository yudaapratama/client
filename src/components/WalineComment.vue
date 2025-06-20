<script setup lang="ts">
/* eslint-disable vue/define-props-declaration */
/* eslint-disable vue/no-unused-properties */
/* eslint-disable vue/require-prop-comment */
/* eslint-disable vue/require-prop-types */
import { useStyleTag } from '@vueuse/core';
import type {
	UpdateCommentResponse,
  WalineComment,
  WalineCommentStatus,
  WalineRootComment,
} from '@waline/api';
import { deleteComment, getComment, updateComment } from '@waline/api';
import { computed, onMounted, onUnmounted, provide, ref, watch } from 'vue';
import * as pkg from "vue-toastification";

import Reaction from './ArticleReaction.vue';
import CommentBox from './CommentBox.vue';
import CommentCard from './CommentCard.vue';
import { LoadingIcon } from './Icons.js';
import { useLikeStorage, useUserInfo } from '../composables/index.js';
import type { WalineCommentSorting, WalineProps } from '../typings/index.js';
import { getConfig, getDarkStyle } from '../utils/index.js';
import { version } from '../version.js';

type SortKey = 'insertedAt_desc' | 'insertedAt_asc' | 'like_desc';
type ResponseLike = UpdateCommentResponse & {
	success?: boolean;
	message?: string;
}

const props = defineProps([
  'serverURL',
  'path',
  'meta',
  'requiredMeta',
  'dark',
  'commentSorting',
  'lang',
  'locale',
  'pageSize',
  'wordLimit',
  'emoji',
  'login',
  'highlighter',
  'texRenderer',
  'imageUploader',
  'search',
  'copyright',
  'recaptchaV3Key',
  'turnstileKey',
  'reaction',
]);

const sortKeyMap: Record<WalineCommentSorting, SortKey> = {
  latest: 'insertedAt_desc',
  oldest: 'insertedAt_asc',
  hottest: 'like_desc',
};
const sortingMethods = Object.keys(sortKeyMap) as WalineCommentSorting[];

const { useToast } = pkg
const toast = useToast();
const userInfo = useUserInfo();
const likeStorage = useLikeStorage();

const status = ref<'loading' | 'success' | 'error'>('loading');

const count = ref(0);
const page = ref(1);
const totalPages = ref(0);

const config = computed(() => getConfig(props as WalineProps));

// eslint-disable-next-line vue/no-ref-object-destructure
const commentSortingRef = ref(config.value.commentSorting);

const data = ref<WalineRootComment[]>([]);
const reply = ref<WalineComment | null>(null);
const edit = ref<WalineComment | null>(null);

const darkmodeStyle = computed(() => getDarkStyle(config.value.dark));

const i18n = computed(() => config.value.locale);

useStyleTag(darkmodeStyle, { id: 'waline-darkmode' });

let abort: () => void;

const getCommentData = (pageNumber: number): void => {
  const { serverURL, path, pageSize } = config.value;
  const controller = new AbortController();

  status.value = 'loading';

  abort?.();

  getComment({
    serverURL,
    lang: config.value.lang,
    path,
    pageSize,
    sortBy: sortKeyMap[commentSortingRef.value],
    page: pageNumber,
    signal: controller.signal,
    token: userInfo.value?.token,
  })
    .then((resp) => {
      status.value = 'success';
      count.value = resp.count;
      data.value.push(...resp.data);
      page.value = pageNumber;
      totalPages.value = resp.totalPages;
    })
    .catch((err: Error) => {
      if (err.name !== 'AbortError') {
        console.error(err.message);
        status.value = 'error';
      }
    });

  abort = controller.abort.bind(controller);
};

const loadMore = (): void => getCommentData(page.value + 1);

const refresh = (): void => {
  count.value = 0;
  data.value = [];
  getCommentData(1);
};

const onSortByChange = (item: WalineCommentSorting): void => {
  if (commentSortingRef.value !== item) {
    commentSortingRef.value = item;
    refresh();
  }
};

const onReply = (comment: WalineComment | null): void => {
  reply.value = comment;
};

const onEdit = (comment: WalineComment | null): void => {
  edit.value = comment;
};

const onSubmit = (comment: WalineComment): void => {
  if (edit.value) {
    edit.value.comment = comment.comment;
    edit.value.orig = comment.orig;
  } else if ('rid' in comment) {
    const repliedComment = data.value.find(
      ({ objectId }) => objectId === comment.rid,
    );

    if (!repliedComment) return;

    if (!Array.isArray(repliedComment.children)) repliedComment.children = [];

    repliedComment.children.push(comment);

		const commentMap = new Map();
		repliedComment.children.forEach(comment => {
			commentMap.set(comment.objectId, comment);
		});
		
		const childrenMap = new Map();
		repliedComment.children.forEach(comment => {
			

			if (!childrenMap.has(comment.pid)) {
				childrenMap.set(comment.pid, []);
			}
			childrenMap.get(comment.pid).push(comment);
		});

		const sorted = [];
		const visited = new Set();

		repliedComment.children.forEach(comment => {
			const isTopLevel = !commentMap.has(comment.pid);
			if (isTopLevel) {
				visit(comment, childrenMap, visited, sorted)
			}
		});

		repliedComment.children = sorted

		if(comment.rid === comment.pid) {
			repliedComment.children.sort((a, b) => b.time - a.time);
		}
		
  } else {
    data.value.unshift(comment);
    count.value += 1;
  }
};

function visit(comment, childrenMap, visited, output) {
  if (visited.has(comment.objectId)) return;
  visited.add(comment.objectId);
  output.push(comment);

  const replies = childrenMap.get(comment.objectId) || [];
  replies.forEach(child => visit(child, childrenMap, visited, output));
	// return output
}

const onStatusChange = async ({
  comment,
  status,
}: {
  comment: WalineComment;
  status: WalineCommentStatus;
}): Promise<void> => {
  if (comment.status === status) return;

  const { serverURL, lang } = config.value;

  await updateComment({
    serverURL,
    lang,
    token: userInfo.value?.token,
    objectId: comment.objectId,
    comment: { status },
  });

  comment.status = status;
};

const onBanned = async ({ user_id }: WalineComment): Promise<void> => {
	try {
		const { serverURL, lang } = config.value;
		const response = await fetch(
			`${serverURL}/api/user/${user_id}?lang=${lang}`,
			{
				method: 'PUT',
				headers: {
					'Content-Type': 'application/json',
					Authorization: `Bearer ${userInfo.value?.token}`,
				},
				body: JSON.stringify({
					is_banned: 1
				}),
			},
		)

		if (response.ok) {
			toast.success('User banned successfully');
			return;
		} else {
			toast.error('Failed to ban user');
		}
		
	} catch (error) {
		console.error(error);
	}
	
}

const onSticky = async (comment: WalineComment): Promise<void> => {
  if ('rid' in comment) return;

  const { serverURL, lang } = config.value;

  await updateComment({
    serverURL,
    lang,
    token: userInfo.value?.token,
    objectId: comment.objectId,
    comment: { sticky: comment.sticky ? 0 : 1 },
  });

  comment.sticky = !comment.sticky;
};

const onDelete = async ({ objectId }: WalineComment): Promise<void> => {
  if (!confirm('Are you sure you want to delete this comment?')) return;

  const { serverURL, lang } = config.value;

  await deleteComment({
    serverURL,
    lang,
    token: userInfo.value?.token,
    objectId,
  });

  // delete comment from data
  data.value.some((item, index) => {
    if (item.objectId === objectId) {
      data.value = data.value.filter((_item, i) => i !== index);

      return true;
    }

    return item.children.some((child, childIndex) => {
      if (child.objectId === objectId) {
        data.value[index].children = item.children.filter(
          (_item, i) => i !== childIndex,
        );

        return true;
      }

      return false;
    });
  });
};

const onLike = async (comment: WalineComment): Promise<void> => {
		const { serverURL, lang } = config.value;
		const { objectId } = comment;
		const hasLiked = likeStorage.value.includes(objectId);
	
		const response: ResponseLike = await updateComment({
			serverURL,
			lang,
			objectId,
			token: userInfo.value?.token,
			comment: { like: !hasLiked },
		});
		
		if (typeof response.errno == 'undefined' && response.success === false) {
			toast?.error(response.message);
			return;
		};
	
		if (hasLiked)
			likeStorage.value = likeStorage.value.filter((id) => id !== objectId);
		else {
			likeStorage.value = [...likeStorage.value, objectId];
	
			if (likeStorage.value.length > 50)
				likeStorage.value = likeStorage.value.slice(-50);
		}
	
		comment.like = (comment.like || 0) + (hasLiked ? -1 : 1);
};

provide('config', config);

onMounted(() => {
  watch(
    // eslint-disable-next-line @typescript-eslint/no-unsafe-return
    () => [props.serverURL, props.path],
    () => refresh(),
    { immediate: true },
  );
});
onUnmounted(() => abort?.());
</script>

<template>
  <div data-waline>
    <Reaction />

    <CommentBox v-if="!reply && !edit" @log="refresh" @submit="onSubmit" />

    <div class="wl-meta-head">
      <div class="wl-count">
        <span v-if="count" class="wl-num" v-text="count" />
        {{ i18n.comment }}
      </div>

      <ul class="wl-sort">
        <li
          v-for="item in sortingMethods"
          :key="item"
          :class="[item === commentSortingRef ? 'active' : '']"
          @click="onSortByChange(item)"
        >
          {{ i18n[item] }}
        </li>
      </ul>
    </div>

    <div class="wl-cards">
      <CommentCard
        v-for="comment in data"
        :key="comment.objectId"
        :root-id="comment.objectId"
        :comment="comment"
        :reply="reply"
        :edit="edit"
        @log="refresh"
        @reply="onReply"
        @edit="onEdit"
        @submit="onSubmit"
        @status="onStatusChange"
        @delete="onDelete"
        @sticky="onSticky"
        @like="onLike"
				@banned="onBanned"
      />
    </div>

    <div v-if="status === 'error'" class="wl-operation">
      <button
        type="button"
        class="wl-btn"
        @click="refresh"
        v-text="i18n.refresh"
      />
    </div>

    <div v-else-if="status === 'loading'" class="wl-loading">
      <LoadingIcon :size="30" />
    </div>

    <div v-else-if="!data.length" class="wl-empty" v-text="i18n.sofa" />

    <!-- Load more button -->
    <div v-else-if="page < totalPages" class="wl-operation">
      <button
        type="button"
        class="wl-btn"
        @click="loadMore"
        v-text="i18n.more"
      />
    </div>

    <!-- Copyright Information -->
    <div v-if="config.copyright" class="wl-power">
      Powered by
      <a
        href="https://github.com/walinejs/waline"
        target="_blank"
        rel="noopener noreferrer"
      >
        Waline
      </a>
      v{{ version }}
    </div>
  </div>
</template>
