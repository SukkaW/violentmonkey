<template>
  <div class="edit frame flex flex-col abs-full" :class="{frozen}">
    <div class="edit-header flex mr-1c">
      <nav>
        <div
          v-for="(label, navKey) in navItems" :key="navKey"
          class="edit-nav-item" :class="{active: nav === navKey}"
          v-text="label"
          @click="nav = navKey"
        />
      </nav>
      <div class="edit-name text-center ellipsis flex-1">
        <span class="subtle" v-if="script.config.removed" v-text="i18n('headerRecycleBin') + ' / '"/>
        {{scriptName}}
      </div>
      <p v-if="frozen && nav === 'code'" v-text="i18n('readonly')"
         class="text-upper text-right text-red"/>
      <div v-else class="edit-hint text-right ellipsis">
        <a :href="externalEditorInfoUrl"
           target="_blank"
           rel="noopener noreferrer"
           v-text="i18n('editHowToHint')"/>
      </div>
      <div class="mr-1">
        <button v-text="i18n('buttonSave')" @click="save"
                v-show="canSave || !frozen" :disabled="!canSave"
                :class="{'has-error': $fe = fatal || errors}" :title="$fe"/>
        <button v-text="i18n('buttonSaveClose')" @click="saveClose"
                v-show="canSave || !frozen" :disabled="!canSave"/>
        <button v-text="i18n('buttonClose')" @click="close"/>
      </div>
    </div>

    <div class="frozen-note shelf mr-2c flex flex-wrap" v-if="frozenNote && nav === 'code'">
      <p v-text="i18n('readonlyNote')"/>
      <keep-alive>
        <VMSettingsUpdate class="flex ml-2c" :script="script"/>
      </keep-alive>
    </div>

    <p v-if="fatal" class="shelf fatal">
      <b v-text="fatal[0]"/>
      {{fatal[1]}}
    </p>

    <vm-code
      class="flex-auto"
      :value="code"
      :readOnly="frozen"
      ref="$code"
      v-show="nav === 'code'"
      :active="nav === 'code'"
      :commands="commands"
      @code-dirty="codeDirty = $event"
    />
    <keep-alive ref="$tabBody">
    <vm-settings
      class="edit-body"
      v-if="nav === 'settings'"
      v-bind="{readOnly, script}"
    />
    <vm-values
      class="edit-body"
      v-else-if="nav === 'values'"
      v-bind="{readOnly, script}"
    />
    <vm-externals
      class="flex-auto"
      v-else-if="nav === 'externals'"
      :value="script"
    />
    <vm-help
      class="edit-body"
      v-else-if="nav === 'help'"
      :hotkeys="hotkeys"
    />
    </keep-alive>

    <div v-if="errors" class="errors shelf my-1c">
      <p v-for="e in errors" :key="e" v-text="e" class="text-red"/>
      <p class="my-1">
        <a :href="urlMatching" target="_blank" rel="noopener noreferrer" v-text="urlMatching"/>
      </p>
    </div>
  </div>
</template>

<script setup>
import { computed, nextTick, onActivated, onDeactivated, onMounted, ref, watch } from 'vue';
import {
  browserWindows,
  debounce, formatByteLength, getScriptName, getScriptUpdateUrl, i18n, isEmpty,
  nullBool2string, sendCmdDirectly, trueJoin,
} from '@/common';
import { deepCopy, deepEqual, objectPick } from '@/common/object';
import { externalEditorInfoUrl, focusMe, showMessage } from '@/common/ui';
import { keyboardService } from '@/common/keyboard';
import VmCode from '@/common/ui/code';
import VmExternals from '@/common/ui/externals';
import options from '@/common/options';
import { getUnloadSentry } from '@/common/router';
import { store } from '../../utils';
import VmSettings from './settings';
import VMSettingsUpdate from './settings-update';
import VmValues from './values';
import VmHelp from './help';

let CM;
let $codeComp;
let disposeList;
let K_SAVE; // deduced from the current CodeMirror keymap
let savedCopy;
let shouldSavePositionOnSave;
let toggleUnloadSentry;

const emit = defineEmits(['close']);
const props = defineProps({
  /** @type {VMScript} */
  initial: Object,
  initialCode: String,
  readOnly: Boolean,
});

const $code = ref();
const $tabBody = ref();
const nav = ref('code');
const canSave = ref(false);
const script = ref();
const code = ref('');
const codeDirty = ref(false);
const commands = {
  save,
  close,
};
const hotkeys = ref();
const errors = ref();
const fatal = ref();
const frozen = ref(false);
const frozenNote = ref(false);
const urlMatching = ref('https://violentmonkey.github.io/api/matching/');

const navItems = computed(() => {
  const { meta, props: { id } } = script.value;
  const req = meta.require.length && '@require';
  const res = !isEmpty(meta.resources) && '@resource';
  const size = store.storageSize;
  return {
    code: i18n('editNavCode'),
    settings: i18n('editNavSettings'),
    ...id && {
      values: i18n('editNavValues') + (size ? ` (${formatByteLength(size)})` : ''),
    },
    ...(req || res) && { externals: [req, res]::trueJoin('/') },
    help: '?',
  };
});
const scriptName = computed(() => (store.title = getScriptName(script.value)));

watch(nav, async val => {
  await nextTick();
  if (val === 'code') CM.focus();
  else focusMe($tabBody.value.$el);
}, { immediate: true });
watch(canSave, val => {
  toggleUnloadSentry(val);
  keyboardService.setContext('canSave', val);
});
// usually errors for resources
watch(() => props.initial.error, error => {
  if (error) {
    showMessage({ text: `${props.initial.message}\n\n${error}` });
  }
});
watch(codeDirty, onDirty);
watch(script, onScript);

const CUSTOM_PROPS = {
  name: '',
  homepageURL: '',
  updateURL: '',
  downloadURL: '',
  origInclude: true,
  origExclude: true,
  origMatch: true,
  origExcludeMatch: true,
};
const toProp = val => val !== '' ? val : null; // `null` removes the prop from script object
const CUSTOM_LISTS = [
  'include',
  'match',
  'exclude',
  'excludeMatch',
];
const toList = text => (
  text.trim()
    ? text.split('\n').map(line => line.trim()).filter(Boolean)
    : null // `null` removes the prop from script object
);
const CUSTOM_ENUM = [
  INJECT_INTO,
  RUN_AT,
];
const toEnum = val => val || null; // `null` removes the prop from script object
const K_PREV_PANEL = 'Alt-PageUp';
const K_NEXT_PANEL = 'Alt-PageDown';
const compareString = (a, b) => (a < b ? -1 : a > b);
/** @param {VMScript.Config} config */
const collectShouldUpdate = ({ shouldUpdate, _editable }) => (
  +shouldUpdate && (shouldUpdate + _editable)
);

{
  // The eslint rule is bugged as this is a block scope, not a global scope.
  const src = props.initial; // eslint-disable-line vue/no-setup-props-destructure
  code.value = props.initialCode; // eslint-disable-line vue/no-setup-props-destructure
  script.value = deepCopy(src);
  watch(() => script.value.config, onChange, { deep: true });
  watch(() => script.value.custom, onChange, { deep: true });
}

onMounted(() => {
  $codeComp = $code.value;
  CM = $codeComp.cm;
  toggleUnloadSentry = getUnloadSentry(null, () => CM.focus());
  if (options.get('editorWindow') && global.history.length === 1) {
    browser.windows?.getCurrent({ populate: true }).then(setupSavePosition);
  }
  store.storageSize = 0;
  // hotkeys
  const navLabels = Object.values(navItems.value);
  const hk = hotkeys.value = [
    [K_PREV_PANEL, ` ${navLabels.join(' < ')}`],
    [K_NEXT_PANEL, ` ${navLabels.join(' > ')}`],
    ...Object.entries($codeComp.expandKeyMap())
    .sort((a, b) => compareString(a[1], b[1]) || compareString(a[0], b[0])),
  ];
  K_SAVE = hk.find(([, cmd]) => cmd === 'save')?.[0];
  if (!K_SAVE) {
    K_SAVE = 'Ctrl-S';
    hk.unshift([K_SAVE, 'save']);
  }
});

onActivated(() => {
  document.body.classList.add('edit-open');
  disposeList = [
    keyboardService.register('a-pageup', switchPrevPanel),
    keyboardService.register('a-pagedown', switchNextPanel),
    keyboardService.register(K_SAVE.replace(/(?:Ctrl|Cmd)-/i, 'ctrlcmd-'), save),
    keyboardService.register('escape', close),
    keyboardService.register('f1', () => { nav.value = 'help'; }),
  ];
  store.title = scriptName.value;
});

onDeactivated(() => {
  document.body.classList.remove('edit-open');
  store.title = null;
  toggleUnloadSentry(false);
  disposeList?.forEach(dispose => dispose());
});

async function save() {
  if (!canSave.value) return;
  if (shouldSavePositionOnSave) savePosition();
  const scr = script.value;
  const { config, custom } = scr;
  const { notifyUpdates } = config;
  const { noframes } = custom;
  try {
    const id = scr.props.id;
    const res = await sendCmdDirectly('ParseScript', {
      id,
      code: $codeComp.getRealContent(),
      config: {
        notifyUpdates: notifyUpdates ? +notifyUpdates : null, // 0, 1, null
        shouldUpdate: collectShouldUpdate(config), // 0, 1, 2
      },
      custom: {
        ...objectPick(custom, Object.keys(CUSTOM_PROPS), toProp),
        ...objectPick(custom, CUSTOM_LISTS, toList),
        ...objectPick(custom, CUSTOM_ENUM, toEnum),
        noframes: noframes ? +noframes : null,
      },
      // User created scripts MUST be marked `isNew` so that
      // the backend is able to check namespace conflicts,
      // otherwise the script with same namespace will be overridden
      isNew: !id,
      message: '',
    });
    const newId = res?.where?.id;
    CM.markClean();
    codeDirty.value = false; // triggers onDirty which sets canSave
    canSave.value = false; // ...and set it explicitly in case codeDirty was false
    frozenNote.value = false;
    errors.value = res.errors;
    script.value = res.update; // triggers onScript+onChange to handle the new `meta` and `props`
    if (newId && !id) history.replaceState(null, scriptName.value, `${ROUTE_SCRIPTS}/${newId}`);
    fatal.value = null;
  } catch (err) {
    fatal.value = err.message.split('\n');
  }
}
function close() {
  if (nav.value !== 'code') {
    nav.value = 'code';
  } else {
    emit('close');
    // FF doesn't emit `blur` when CodeMirror's textarea is removed
    if (IS_FIREFOX) document.activeElement?.blur();
  }
}
function saveClose() {
  save().then(close);
}
function switchPanel(step) {
  const keys = Object.keys(navItems.value);
  nav.value = keys[(keys.indexOf(nav.value) + step + keys.length) % keys.length];
}
function switchPrevPanel() {
  switchPanel(-1);
}
function switchNextPanel() {
  switchPanel(1);
}
function onChange(evt) {
  const scr = script.value;
  const { config } = scr;
  const { removed } = config;
  const remote = scr._remote = !!getScriptUpdateUrl(scr);
  const remoteMode = remote && collectShouldUpdate(config);
  const fz = !!(removed || remoteMode === 1 || props.readOnly);
  frozen.value = fz;
  frozenNote.value = !removed && (fz || remoteMode >= 1);
  if (!removed && evt) onDirty();
}
function onDirty() {
  canSave.value = codeDirty.value || !deepEqual(script.value, savedCopy);
}
function onScript(scr) {
  const { custom, config } = scr;
  const { shouldUpdate } = config;
  // Matching Vue model types, so deepEqual can work properly
  config._editable = shouldUpdate === 2;
  config.shouldUpdate = !!shouldUpdate;
  config.notifyUpdates = nullBool2string(config.notifyUpdates);
  custom.noframes = nullBool2string(custom.noframes);
  // Adding placeholders for any missing values so deepEqual can work properly
  for (const key in CUSTOM_PROPS) {
    if (custom[key] == null) custom[key] = CUSTOM_PROPS[key];
  }
  for (const key of CUSTOM_ENUM) {
    if (!custom[key]) custom[key] = '';
  }
  for (const key of CUSTOM_LISTS) {
    const val = custom[key];
    // Adding a new row so the user can click it and type, just like in an empty textarea.
    custom[key] = val ? `${val.join('\n')}${val.length ? '\n' : ''}` : '';
  }
  onChange();
  if (!config.removed) savedCopy = deepCopy(scr);
}
/** @param {chrome.windows.Window} [wnd] */
async function savePosition(wnd) {
  if (options.get('editorWindow')) {
    if (!wnd) wnd = await browserWindows?.getCurrent() || {};
    /* chrome.windows API can't set both the state and coords, so we have to choose:
     * either we save the min/max state and lose the coords on restore,
     * or we lose the min/max state and save the normal coords.
     * Let's assume those who use a window prefer it at a certain position most of the time,
     * and occasionally minimize/maximize it, but wouldn't want to save the state. */
    if (wnd.state === 'normal') {
      options.set('editorWindowPos', objectPick(wnd, ['left', 'top', 'width', 'height']));
    }
  }
}

/** @param {chrome.windows.Window} _ */
function setupSavePosition({ id: curWndId, tabs }) {
  if (tabs.length === 1) {
    const { onBoundsChanged } = chrome.windows;
    if (onBoundsChanged) {
      // triggered on moving/resizing, Chrome 86+
      onBoundsChanged.addListener(wnd => {
        if (wnd.id === curWndId) savePosition(wnd);
      });
    } else {
      // triggered on resizing only
      addEventListener('resize', debounce(savePosition, 100));
      shouldSavePositionOnSave = true;
    }
  }
}
</script>

<style>
.edit {
  --border: 1px solid var(--fill-3);
  z-index: 2000;
  &-header {
    position: sticky;
    top: 0;
    z-index: 1;
    align-items: center;
    justify-content: space-between;
    border-bottom: var(--border);
    background: inherit;
  }
  &-name {
    font-weight: bold;
  }
  &-body {
    padding: .5rem 1rem;
    // overflow: auto;
    background: var(--bg);
  }
  &-nav-item {
    display: inline-block;
    padding: 8px 16px;
    cursor: pointer;
    &.active {
      background: var(--bg);
      box-shadow: 0 -1px 1px var(--fill-7);
    }
    &:not(.active):hover {
      background: var(--fill-0-5);
      box-shadow: 0 -1px 1px var(--fill-4);
    }
  }
  .edit-externals {
    --border: 0;
    .select {
      padding-top: 0.5em;
      @media (max-width: 1599px) {
        resize: vertical;
        &[style*=height] {
          max-height: 80%;
        }
        &[style*=width] {
          width: auto !important;
        }
      }
    }
    @media (min-width: 1600px) {
      flex-direction: row;
      .select {
        resize: horizontal;
        min-width: 15em;
        width: 30%;
        max-height: none;
        border-bottom: none;
        &[style*=height] {
          height: auto !important;
        }
        &[style*=width] {
          max-width: 80%;
        }
      }
    }
  }
  .errors {
    --border: none;
    border-top: 2px solid red;
  }
  .fatal {
    background: firebrick;
    color: white;
  }
  .frozen-note {
    background: var(--bg);
  }
  .shelf {
    padding: .5em 1em;
    border-bottom: var(--border);
  }
  &.frozen .CodeMirror {
    background: var(--fill-0-5);
  }
}

.touch .edit {
  // fixed/absolute doesn't work well with scroll in Firefox Android
  position: static;
  // larger than 100vh to force overflow so that the toolbar can be hidden in Firefox Android
  min-height: calc(100vh + 1px);
}

.edit-open {
  /* TODO: fix vueleton's tooltip to not destroy layout at right bottom window corner */
  overflow: hidden;
}

@media (max-width: 767px) {
  .edit-hint {
    display: none;
  }
}

@media (max-width: 500px) {
  .edit-name {
    display: none;
  }
}
</style>
