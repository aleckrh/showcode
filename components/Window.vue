<template>
    <div
        ref="root"
        class="relative"
        :class="[
            {
                'origin-center': settings.position === 'center',
                'origin-top': settings.position === 'top',
                'origin-bottom': settings.position === 'bottom',
                'origin-left': settings.position === 'left',
                'origin-right': settings.position === 'right',
            },
        ]"
        :style="{
            border: border,
            borderRadius: borderRadius,
            boxShadow: boxShadowWithAccent,
            fontSize: `${settings.fontSize}px`,
            transform: `scale(${settings.scale})`,
            lineHeight: `${settings.lineHeight}px`,
            marginTop: `${settings.marginTop}px`,
            marginBottom: `${settings.marginBottom}px`,
            marginLeft: `${settings.marginLeft}px`,
            marginRight: `${settings.marginRight}px`,
            backgroundColor: settings.themeBackground,
        }"
    >
        <Interact v-if="!preview" drag @dragmove="$emit('update:scale', $event.delta.y)">
            <ButtonResize
                data-hide
                :zoom-scale="Math.pow(settings.scale * zoom, -1)"
                class="bottom-0 left-1/2 -m-1.5 w-2.5 h-2.5 invisible group-hover:visible absolute"
            />
        </Interact>

        <div
            v-if="settings.showHeader"
            class="relative flex items-center h-12 p-4 exclude-from-panzoom"
            :style="
                settings.showHeaderAccent
                    ? {
                          borderColor: borderColor,
                          backgroundColor: backgroundAccentColor,
                          borderTopRightRadius: headerBorderRadiusTopRight,
                          borderTopLeftRadius: headerBorderRadiusTopLeft,
                      }
                    : {}
            "
        >
            <FauxMenu
                v-if="settings.showMenu"
                class="absolute"
                :theme-background="settings.themeBackground"
                :theme="settings.showColorMenu ? 'color' : settings.themeType"
            />

            <div
                v-if="settings.showTitle"
                @click="preview ? null : editTitle()"
                class="w-full px-2 text-center text-gray-400 whitespace-nowrap"
                :class="{
                    'mx-14': settings.showMenu,
                    'hover:ring hover:ring-ui-violet-500 hover:rounded-lg cursor-text': !preview,
                }"
            >
                <input
                    v-if="editingTitle || title.length > 0"
                    type="text"
                    ref="titleInput"
                    v-model="title"
                    :readonly="preview"
                    @blur="editingTitle = false"
                    @keyup.enter="$refs.titleInput.blur()"
                    :style="{ width: `${title.length / 1.75}em` }"
                    :class="{ 'cursor-pointer pointer-events-none': preview }"
                    class="p-0 text-sm font-medium text-center bg-transparent border-0 shadow-none focus:ring-0"
                />

                <span v-else class="text-sm font-medium"> Untitled-1 </span>
            </div>
        </div>

        <div
            :class="[
                {
                    flex: settings.landscape && blocks.length > 1,
                    'flex flex-col': !settings.landscape && blocks.length > 1,
                    'divide-x': settings.landscape && settings.showDividers,
                    'divide-y': !settings.landscape && settings.showDividers,
                },
            ]"
            :style="{ borderColor: borderColor }"
        >
            <div
                class="flex items-center overflow-hidden"
                v-for="({ lines, added, removed, focused }, index) in blocks"
                :key="index"
                :style="{
                    borderColor: borderColor,
                    paddingTop: `${padding('top')}px`,
                    paddingBottom: `${padding('bottom')}px`,
                }"
            >
                <Code
                    class="relative w-full"
                    v-bind="codeAttributes"
                    :lines="lines"
                    :added="added"
                    :removed="removed"
                    :focused="focused"
                    :theme-type="settings.themeType"
                    :show-line-numbers="settings.showLineNumbers"
                />
            </div>
        </div>

        <div
            v-if="settings.showSocialBadge"
            class="absolute w-full flex"
            :class="{
                'justify-start': settings.socialPosition === 'bottom-left',
                'justify-center': settings.socialPosition === 'bottom-center',
                'justify-end': settings.socialPosition === 'bottom-right',
            }"
        >
            <div
                :style="{
                    color: fontColor,
                    borderColor: borderColor,
                    boxShadow: boxShadowWithAccent,
                    backgroundColor: settings.themeBackground,
                }"
                class="p-1 mt-4 flex items-center rounded-full border"
            >
                <SocialBadgeIcon class="w-8" :type="settings.socialType" />

                <div class="px-2 space-y-0.5">
                    <div
                        v-if="settings.socialDisplayName"
                        class="text-xs font-semibold leading-none"
                    >
                        {{ settings.socialDisplayName }}
                    </div>

                    <div
                        v-if="settings.socialUsername"
                        class="font-medium leading-none text-[11px] opacity-75"
                    >
                        @{{ settings.socialUsername }}
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
import chroma from 'chroma-js';
import useFonts from '@/composables/useFonts';
import { get, merge, cloneDeep, capitalize } from 'lodash';
import { ref, watch, nextTick, computed } from '@nuxtjs/composition-api';

export default {
    props: {
        zoom: {
            type: [Number, String],
            required: false,
            default: 1,
        },
        blocks: {
            type: Array,
            default: () => [],
        },
        preview: {
            type: Boolean,
            default: false,
        },
        settings: {
            type: Object,
            default: () => {},
        },
    },

    setup(props, { emit }) {
        const { fontFamilies } = useFonts();

        const root = ref(null);
        const titleInput = ref(null);
        const editingTitle = ref(false);
        const title = ref(String(props.settings.title || ''));

        function editTitle() {
            editingTitle.value = true;

            nextTick(() => titleInput.value.focus());
        }

        const fontAttributes = computed(() => {
            const font = fontFamilies.value.find((font) => font.name === props.settings.fontFamily);

            return get(font, 'attributes', { class: 'font-mono' });
        });

        const fontColor = computed(() => {
            const color = chroma(props.settings.themeBackground);

            if (props.settings.themeType === 'light') {
                return color.darken(4).hex();
            }

            return color.brighten(4).hex();
        });

        const codeAttributes = computed(() => {
            return merge(cloneDeep(fontAttributes.value), {
                style: {
                    paddingLeft: `${padding('left')}px`,
                    paddingRight: `${padding('right')}px`,
                },
            });
        });

        const borderColor = computed(() => {
            return chroma(props.settings.themeBackground)
                .darken(props.settings.themeType === 'light' ? 1 : -3)
                .alpha(0.25)
                .hex();
        });

        const borderHighlight = computed(() => {
            return {
                dark: chroma(props.settings.themeBackground).brighten(3).alpha(0.5).hex(),
                light: chroma(props.settings.themeBackground).darken(2).alpha(1).hex(),
            }[props.settings.themeType];
        });

        const boxShadowAccent = computed(() => {
            if (props.settings.showBorder) {
                return;
            }

            return {
                dark: `0 0 0 1px ${borderHighlight.value},0 0 0 1.5px ${props.settings.themeBackground}`,
                light: `0 0 0 0.5px ${borderHighlight.value}`,
            }[props.settings.themeType];
        });

        const backgroundAccentColor = computed(() => {
            return chroma(props.settings.themeBackground)
                .darken(props.settings.themeType === 'light' ? 1 : -3)
                .alpha(0.1)
                .hex();
        });

        const boxShadow = computed(() => {
            if (!props.settings.showShadow || !props.settings.shadowColor) {
                return;
            }

            const color = [
                props.settings.shadowColor.red,
                props.settings.shadowColor.green,
                props.settings.shadowColor.blue,
                props.settings.shadowColor.alpha,
            ].join(', ');

            const shadow = [
                props.settings.shadowX,
                props.settings.shadowY,
                props.settings.shadowBlur,
                props.settings.shadowSpread,
            ]
                .map((prop) => prop + 'px')
                .join(' ');

            return [shadow, `rgba(${color})`].join(' ');
        });

        const boxShadowWithAccent = computed(() => {
            return [boxShadow.value, boxShadowAccent.value].filter((value) => value).join(',');
        });

        const headerBorderRadiusTopLeft = computed(() => {
            const radius = props.settings.borderRadiusLocked
                ? props.settings.borderRadius
                : props.settings.borderRadiusTopLeft;

            return `${radius - props.settings.borderWidth}px`;
        });

        const headerBorderRadiusTopRight = computed(() => {
            const radius = props.settings.borderRadiusLocked
                ? props.settings.borderRadius
                : props.settings.borderRadiusTopRight;

            return `${radius - props.settings.borderWidth}px`;
        });

        const borderRadius = computed(() => {
            if (props.settings.borderRadiusLocked) {
                return `${props.settings.borderRadius}px`;
            }

            return [
                props.settings.borderRadiusTopLeft,
                props.settings.borderRadiusTopRight,
                props.settings.borderRadiusBottomRight,
                props.settings.borderRadiusBottomLeft,
            ]
                .map((radius) => radius + 'px')
                .join(' ');
        });

        const border = computed(() => {
            if (!props.settings.showBorder || !props.settings.borderColor) {
                return;
            }

            const color = [
                props.settings.borderColor.red,
                props.settings.borderColor.green,
                props.settings.borderColor.blue,
                props.settings.borderColor.alpha,
            ].join(', ');

            return `${props.settings.borderWidth}px solid rgba(${color})`;
        });

        const actualWidth = () => root.value.clientWidth;
        const actualHeight = () => root.value.clientHeight;

        const padding = (side) =>
            props.settings.paddingLocked
                ? props.settings.padding
                : get(props.settings, `padding${capitalize(side)}`);

        watch(title, (title) => emit('update:title', title));

        watch(
            () => props.settings.title,
            (newTitle) => (title.value = newTitle)
        );

        return {
            root,
            title,
            editTitle,
            editingTitle,
            titleInput,
            border,
            padding,
            boxShadowWithAccent,
            borderColor,
            borderRadius,
            headerBorderRadiusTopLeft,
            headerBorderRadiusTopRight,
            backgroundAccentColor,
            actualWidth,
            actualHeight,
            codeAttributes,
            fontColor,
        };
    },
};
</script>
