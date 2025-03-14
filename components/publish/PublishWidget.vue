<script setup lang="ts">
import { EditorContent } from '@tiptap/vue-3'
import stringLength from 'string-length'
import type { mastodon } from 'masto'
import type { Draft } from '~/types'

const {
  draftKey,
  initial = getDefaultDraft,
  expanded = false,
  placeholder,
  dialogLabelledBy,
} = defineProps<{
  draftKey?: string
  initial?: () => Draft
  placeholder?: string
  inReplyToId?: string
  inReplyToVisibility?: mastodon.v1.StatusVisibility
  expanded?: boolean
  dialogLabelledBy?: string
}>()

const emit = defineEmits<{
  (evt: 'published', status: mastodon.v1.Status): void
}>()

const { t } = useI18n()

const draftState = useDraft(draftKey, initial)
const { draft } = $(draftState)

const {
  isExceedingAttachmentLimit, isUploading, failedAttachments, isOverDropZone,
  uploadAttachments, pickAttachments, setDescription, removeAttachment,
  dropZoneRef,
} = $(useUploadMediaAttachment($$(draft)))

let { shouldExpanded, isExpanded, isSending, isPublishDisabled, publishDraft, failedMessages, preferredLanguage, publishSpoilerText } = $(usePublish(
  {
    draftState,
    ...$$({ expanded, isUploading, initialDraft: initial }),
  },
))

const { editor } = useTiptap({
  content: computed({
    get: () => draft.params.status,
    set: (newVal) => {
      draft.params.status = newVal
      draft.lastUpdated = Date.now()
    },
  }),
  placeholder: computed(() => placeholder ?? draft.params.inReplyToId ? t('placeholder.replying') : t('placeholder.default_1')),
  autofocus: shouldExpanded,
  onSubmit: publish,
  onFocus() {
    if (!isExpanded && draft.initialText) {
      editor.value?.chain().insertContent(`${draft.initialText} `).focus('end').run()
      draft.initialText = ''
    }
    isExpanded = true
  },
  onPaste: handlePaste,
})

function trimPollOptions() {
  const indexLastNonEmpty = draft.params.poll!.options.findLastIndex(option => option.trim().length > 0)
  const trimmedOptions = draft.params.poll!.options.slice(0, indexLastNonEmpty + 1)

  if (currentInstance.value?.configuration
      && trimmedOptions.length >= currentInstance.value?.configuration?.polls.maxOptions)
    draft.params.poll!.options = trimmedOptions
  else
    draft.params.poll!.options = [...trimmedOptions, '']
}

function editPollOptionDraft(event: Event, index: number) {
  draft.params.poll!.options[index] = (event.target as HTMLInputElement).value
  trimPollOptions()
}

function deletePollOption(index: number) {
  draft.params.poll!.options.splice(index, 1)
  trimPollOptions()
}

const expiresInOptions = [
  {
    seconds: 1 * 60 * 60,
    label: t('time_ago_options.hour_future', 1),
  },
  {
    seconds: 2 * 60 * 60,
    label: t('time_ago_options.hour_future', 2),
  },
  {
    seconds: 1 * 24 * 60 * 60,
    label: t('time_ago_options.day_future', 1),
  },
  {
    seconds: 2 * 24 * 60 * 60,
    label: t('time_ago_options.day_future', 2),
  },
  {
    seconds: 7 * 24 * 60 * 60,
    label: t('time_ago_options.day_future', 7),
  },
]

const expiresInDefaultOptionIndex = 2

const characterCount = $computed(() => {
  const text = htmlToText(editor.value?.getHTML() || '')

  let length = stringLength(text)

  // taken from https://github.com/mastodon/mastodon/blob/07f8b4d1b19f734d04e69daeb4c3421ef9767aac/app/lib/text_formatter.rb
  const linkRegex = /(https?:\/\/(www\.)?|xmpp:)\S+/g

  // taken from https://github.com/mastodon/mastodon/blob/af578e/app/javascript/mastodon/features/compose/util/counter.js
  const countableMentionRegex = /(^|[^/\w])@(([a-z0-9_]+)@[a-z0-9.-]+[a-z0-9]+)/ig

  // maximum of 23 chars per link
  // https://github.com/elk-zone/elk/issues/1651
  const maxLength = 23

  for (const [fullMatch] of text.matchAll(linkRegex))
    length -= fullMatch.length - Math.min(maxLength, fullMatch.length)

  for (const [fullMatch, before, _handle, username] of text.matchAll(countableMentionRegex))
    length -= fullMatch.length - (before + username).length - 1 // - 1 for the @

  if (draft.mentions) {
    // + 1 is needed as mentions always need a space seperator at the end
    length += draft.mentions.map((mention) => {
      const [handle] = mention.split('@')
      return `@${handle}`
    }).join(' ').length + 1
  }

  length += stringLength(publishSpoilerText)

  return length
})

const isExceedingCharacterLimit = $computed(() => {
  return characterCount > characterLimit.value
})

const postLanguageDisplay = $computed(() => languagesNameList.find(i => i.code === (draft.params.language || preferredLanguage))?.nativeName)

async function handlePaste(evt: ClipboardEvent) {
  const files = evt.clipboardData?.files
  if (!files || files.length === 0)
    return

  evt.preventDefault()
  await uploadAttachments(Array.from(files))
}

function insertEmoji(name: string) {
  editor.value?.chain().focus().insertEmoji(name).run()
}
function insertCustomEmoji(image: any) {
  editor.value?.chain().focus().insertCustomEmoji(image).run()
}

async function toggleSensitive() {
  draft.params.sensitive = !draft.params.sensitive
}

async function publish() {
  const status = await publishDraft()
  if (status)
    emit('published', status)
}

useWebShareTarget(async ({ data: { data, action } }: any) => {
  if (action !== 'compose-with-shared-data')
    return

  editor.value?.commands.focus('end')

  for (const text of data.textParts) {
    for (const line of text.split('\n')) {
      editor.value?.commands.insertContent({
        type: 'paragraph',
        content: [{ type: 'text', text: line }],
      })
    }
  }

  if (data.files.length !== 0)
    await uploadAttachments(data.files)
})

defineExpose({
  focusEditor: () => {
    editor.value?.commands?.focus?.()
  },
})

onDeactivated(() => {
  clearEmptyDrafts()
})
</script>

<template>
  <div v-if="isHydrated && currentUser" flex="~ col gap-4" py3 px2 sm:px4 aria-roledescription="publish-widget">
    <template v-if="draft.editingStatus">
      <div flex="~ col gap-1">
        <div id="state-editing" text-secondary self-center>
          {{ $t('state.editing') }}
        </div>
        <StatusCard :status="draft.editingStatus" :actions="false" :hover="false" is-preview px-0 />
      </div>
      <div border="b dashed gray/40" />
    </template>

    <div flex gap-3 flex-1>
      <NuxtLink :to="getAccountRoute(currentUser.account)">
        <AccountBigAvatar :account="currentUser.account" square />
      </NuxtLink>
      <!-- This `w-0` style is used to avoid overflow problems in flex layouts，so don't remove it unless you know what you're doing -->
      <div
        ref="dropZoneRef"
        flex w-0 flex-col gap-3 flex-1
        border="2 dashed transparent"
        :class="[isSending ? 'pointer-events-none' : '', isOverDropZone ? '!border-primary' : '']"
      >
        <ContentMentionGroup v-if="draft.mentions?.length && shouldExpanded" replying>
          <button v-for="m, i of draft.mentions" :key="m" text-primary hover:color-red @click="draft.mentions?.splice(i, 1)">
            {{ accountToShortHandle(m) }}
          </button>
        </ContentMentionGroup>

        <div v-if="draft.params.sensitive">
          <input
            v-model="publishSpoilerText"
            type="text"
            :placeholder="$t('placeholder.content_warning')"
            p2 border-rounded w-full bg-transparent
            outline-none border="~ base"
          >
        </div>

        <CommonErrorMessage v-if="failedMessages.length > 0" described-by="publish-failed">
          <header id="publish-failed" flex justify-between>
            <div flex items-center gap-x-2 font-bold>
              <div aria-hidden="true" i-ri:error-warning-fill />
              <p>{{ $t('state.publish_failed') }}</p>
            </div>
            <CommonTooltip placement="bottom" :content="$t('action.clear_publish_failed')">
              <button
                flex rounded-4 p1 hover:bg-active cursor-pointer transition-100 :aria-label="$t('action.clear_publish_failed')"
                @click="failedMessages = []"
              >
                <span aria-hidden="true" w="1.75em" h="1.75em" i-ri:close-line />
              </button>
            </CommonTooltip>
          </header>
          <ol ps-2 sm:ps-1>
            <li v-for="(error, i) in failedMessages" :key="i" flex="~ col sm:row" gap-y-1 sm:gap-x-2>
              <strong>{{ i + 1 }}.</strong>
              <span>{{ error }}</span>
            </li>
          </ol>
        </CommonErrorMessage>

        <div relative flex-1 flex flex-col>
          <EditorContent
            :editor="editor"
            flex max-w-full
            :class="shouldExpanded ? 'min-h-30 md:max-h-[calc(100vh-200px)] sm:max-h-[calc(100vh-400px)] max-h-35 of-y-auto overscroll-contain' : ''"
          />
        </div>

        <div v-if="isUploading" flex gap-1 items-center text-sm p1 text-primary>
          <div animate-spin preserve-3d>
            <div i-ri:loader-2-fill />
          </div>
          {{ $t('state.uploading') }}
        </div>
        <CommonErrorMessage
          v-else-if="failedAttachments.length > 0"
          :described-by="isExceedingAttachmentLimit ? 'upload-failed uploads-per-post' : 'upload-failed'"
        >
          <header id="upload-failed" flex justify-between>
            <div flex items-center gap-x-2 font-bold>
              <div aria-hidden="true" i-ri:error-warning-fill />
              <p>{{ $t('state.upload_failed') }}</p>
            </div>
            <CommonTooltip placement="bottom" :content="$t('action.clear_upload_failed')">
              <button
                flex rounded-4 p1 hover:bg-active cursor-pointer transition-100
                :aria-label="$t('action.clear_upload_failed')" @click="failedAttachments = []"
              >
                <span aria-hidden="true" w="1.75em" h="1.75em" i-ri:close-line />
              </button>
            </CommonTooltip>
          </header>
          <div v-if="isExceedingAttachmentLimit" id="uploads-per-post" ps-2 sm:ps-1 text-small>
            {{ $t('state.attachments_exceed_server_limit') }}
          </div>
          <ol ps-2 sm:ps-1>
            <li v-for="error in failedAttachments" :key="error[0]" flex="~ col sm:row" gap-y-1 sm:gap-x-2>
              <strong>{{ error[1] }}:</strong>
              <span>{{ error[0] }}</span>
            </li>
          </ol>
        </CommonErrorMessage>

        <div v-if="draft.attachments.length" flex="~ col gap-2" overflow-auto>
          <PublishAttachment
            v-for="(att, idx) in draft.attachments" :key="att.id"
            :attachment="att"
            :dialog-labelled-by="dialogLabelledBy ?? (draft.editingStatus ? 'state-editing' : undefined)"
            @remove="removeAttachment(idx)"
            @set-description="setDescription(att, $event)"
          />
        </div>
      </div>
    </div>
    <div flex gap-4>
      <div w-12 h-full sm:block hidden />
      <div flex="~ col 1" max-w-full>
        <form v-if="isExpanded && draft.params.poll" my-4 flex="~ 1 col" gap-3 m="s--1">
          <div
            v-for="(option, index) in draft.params.poll.options"
            :key="index"
            flex="~ row"
            gap-3
          >
            <input
              :value="option"
              bg-base
              border="~ base" flex-1 h10 pe-4 rounded-2 w-full flex="~ row"
              items-center relative focus-within:box-shadow-outline gap-3
              px-4 py-2
              :placeholder="$t('polls.option_placeholder', { current: index + 1, max: currentInstance?.configuration?.polls.maxOptions })"
              class="option-input"
              @input="editPollOptionDraft($event, index)"
            >
            <CommonTooltip placement="top" :content="$t('polls.remove_option')" class="delete-button">
              <button
                btn-action-icon class="hover:bg-red/75"
                :disabled="index === draft.params.poll!.options.length - 1 && (index + 1 !== currentInstance?.configuration?.polls.maxOptions || draft.params.poll!.options[index].length === 0)"
                @click.prevent="deletePollOption(index)"
              >
                <div i-ri:delete-bin-line />
              </button>
            </CommonTooltip>
            <span
              v-if="currentInstance?.configuration?.polls.maxCharactersPerOption"
              class="char-limit-radial"
              aspect-ratio-1
              h-10
              :style="{ background: `radial-gradient(closest-side, rgba(var(--rgb-bg-base)) 79%, transparent 80% 100%), conic-gradient(${draft.params.poll!.options[index].length / currentInstance?.configuration?.polls.maxCharactersPerOption > 1 ? 'var(--c-danger)' : 'var(--c-primary)'} ${draft.params.poll!.options[index].length / currentInstance?.configuration?.polls.maxCharactersPerOption * 100}%, var(--c-primary-fade) 0)` }"
            >{{ draft.params.poll!.options[index].length }}</span>
          </div>
        </form>
        <div
          v-if="shouldExpanded" flex="~ gap-1 1 wrap" m="s--1" pt-2 justify="end" max-w-full
          border="t base"
        >
          <PublishEmojiPicker
            @select="insertEmoji"
            @select-custom="insertCustomEmoji"
          >
            <button btn-action-icon :title="$t('tooltip.emoji')">
              <div i-ri:emotion-line />
            </button>
          </PublishEmojiPicker>

          <CommonTooltip v-if="draft.params.poll === undefined" placement="top" :content="$t('tooltip.add_media')">
            <button btn-action-icon :aria-label="$t('tooltip.add_media')" @click="pickAttachments">
              <div i-ri:image-add-line />
            </button>
          </CommonTooltip>

          <template v-if="draft.attachments.length === 0">
            <CommonTooltip v-if="!draft.params.poll" placement="top" :content="$t('polls.create')">
              <button btn-action-icon :aria-label="$t('polls.create')" @click="draft.params.poll = { options: [''], expiresIn: expiresInOptions[expiresInDefaultOptionIndex].seconds }">
                <div i-ri:chat-poll-line />
              </button>
            </CommonTooltip>
            <div v-else rounded-full b-1 border-dark flex="~ row" gap-1>
              <CommonTooltip placement="top" :content="$t('polls.cancel')">
                <button btn-action-icon b-r border-dark :aria-label="$t('polls.cancel')" @click="draft.params.poll = undefined">
                  <div i-ri:close-line />
                </button>
              </CommonTooltip>
              <CommonDropdown placement="top">
                <CommonTooltip placement="top" :content="$t('polls.settings')">
                  <button :aria-label="$t('polls.settings')" btn-action-icon w-12>
                    <div i-ri:list-settings-line />
                    <div i-ri:arrow-down-s-line text-sm text-secondary me--1 />
                  </button>
                </CommonTooltip>
                <template #popper>
                  <div flex="~ col" gap-1 p-2>
                    <CommonCheckbox v-model="draft.params.poll.multiple" :label="draft.params.poll.multiple ? $t('polls.disallow_multiple') : $t('polls.allow_multiple')" px-2 gap-3 h-9 flex justify-center hover:bg-active rounded-full icon-checked="i-ri:checkbox-multiple-blank-line" icon-unchecked="i-ri:checkbox-blank-circle-line" />
                    <CommonCheckbox v-model="draft.params.poll.hideTotals" :label="draft.params.poll.hideTotals ? $t('polls.show_votes') : $t('polls.hide_votes')" px-2 gap-3 h-9 flex justify-center hover:bg-active rounded-full icon-checked="i-ri:eye-close-line" icon-unchecked="i-ri:eye-line" />
                  </div>
                </template>
              </CommonDropdown>
              <CommonDropdown placement="bottom">
                <CommonTooltip placement="top" :content="$t('polls.expiration')">
                  <button :aria-label="$t('polls.expiration')" btn-action-icon w-12>
                    <div i-ri:hourglass-line />
                    <div i-ri:arrow-down-s-line text-sm text-secondary me--1 />
                  </button>
                </CommonTooltip>
                <template #popper>
                  <CommonDropdownItem
                    v-for="expiresInOption in expiresInOptions"
                    :key="expiresInOption.seconds"
                    :text="expiresInOption.label"
                    :checked="draft.params.poll!.expiresIn === expiresInOption.seconds"
                    @click="draft.params.poll!.expiresIn = expiresInOption.seconds"
                  />
                </template>
              </CommonDropdown>
            </div>
          </template>

          <PublishEditorTools v-if="editor" :editor="editor" />

          <div flex-auto />

          <PublishCharacterCounter :max="characterLimit" :length="characterCount" />

          <CommonTooltip placement="top" :content="$t('tooltip.change_language')">
            <CommonDropdown placement="bottom" auto-boundary-max-size>
              <button btn-action-icon :aria-label="$t('tooltip.change_language')" w-max mr1>
                <span v-if="postLanguageDisplay" text-secondary text-sm ml1>{{ postLanguageDisplay }}</span>
                <div v-else i-ri:translate-2 />
                <div i-ri:arrow-down-s-line text-sm text-secondary me--1 />
              </button>

              <template #popper>
                <PublishLanguagePicker v-model="draft.params.language" min-w-80 />
              </template>
            </CommonDropdown>
          </CommonTooltip>

          <CommonTooltip placement="top" :content="$t('tooltip.add_content_warning')">
            <button btn-action-icon :aria-label="$t('tooltip.add_content_warning')" @click="toggleSensitive">
              <div v-if="draft.params.sensitive" i-ri:alarm-warning-fill text-orange />
              <div v-else i-ri:alarm-warning-line />
            </button>
          </CommonTooltip>

          <PublishVisibilityPicker v-model="draft.params.visibility" :editing="!!draft.editingStatus">
            <template #default="{ visibility }">
              <button :disabled="!!draft.editingStatus" :aria-label="$t('tooltip.change_content_visibility')" btn-action-icon :class="{ 'w-12': !draft.editingStatus }">
                <div :class="visibility.icon" />
                <div v-if="!draft.editingStatus" i-ri:arrow-down-s-line text-sm text-secondary me--1 />
              </button>
            </template>
          </PublishVisibilityPicker>

          <CommonTooltip v-if="failedMessages.length > 0" id="publish-failed-tooltip" placement="top" :content="$t('tooltip.publish_failed')">
            <button
              btn-danger rounded-3 text-sm w-full flex="~ gap1" items-center md:w-fit aria-describedby="publish-failed-tooltip"
            >
              <span block>
                <div block i-carbon:face-dizzy-filled />
              </span>
              <span>{{ $t('state.publish_failed') }}</span>
            </button>
          </CommonTooltip>

          <CommonTooltip v-else id="publish-tooltip" placement="top" :content="$t('tooltip.add_publishable_content')" :disabled="!(isPublishDisabled || isExceedingCharacterLimit)">
            <button
              btn-solid rounded-3 text-sm w-full flex="~ gap1" items-center
              md:w-fit
              class="publish-button"
              :aria-disabled="isPublishDisabled || isExceedingCharacterLimit"
              aria-describedby="publish-tooltip"
              @click="publish"
            >
              <span v-if="isSending" block animate-spin preserve-3d>
                <div block i-ri:loader-2-fill />
              </span>
              <span v-if="failedMessages.length" block>
                <div block i-carbon:face-dizzy-filled />
              </span>
              <span v-if="draft.editingStatus">{{ $t('action.save_changes') }}</span>
              <span v-else-if="draft.params.inReplyToId">{{ $t('action.reply') }}</span>
              <span v-else>{{ !isSending ? $t('action.publish') : $t('state.publishing') }}</span>
            </button>
          </CommonTooltip>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
  .publish-button[aria-disabled=true] {
    cursor: not-allowed;
    background-color: var(--c-bg-btn-disabled);
    color: var(--c-text-btn-disabled);
  }
  .publish-button[aria-disabled=true]:hover {
    background-color: var(--c-bg-btn-disabled);
    color: var(--c-text-btn-disabled);
  }
  .option-input:focus + .delete-button {
    display: none;
  }

  .option-input:not(:focus) + .delete-button + .char-limit-radial {
    display: none;
  }

  .char-limit-radial {
    display: flex;
    justify-content: center;
    align-items: center;
    border-radius: 50%;
  }
</style>
