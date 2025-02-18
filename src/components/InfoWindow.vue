<template>
  <div v-if="hasSlotContent" class="info-window-wrapper">
    <div ref="infoWindowRef" v-bind="$attrs">
      <slot />
    </div>
  </div>
</template>

<script lang="ts">
import {
  defineComponent,
  PropType,
  watch,
  ref,
  computed,
  inject,
  onBeforeUnmount,
  Comment,
  onMounted,
  markRaw,
} from "vue";
import equal from "fast-deep-equal";
import { apiSymbol, mapSymbol, markerSymbol } from "../shared/index";

const infoWindowEvents = ["closeclick", "content_changed", "domready", "position_changed", "visible", "zindex_changed"];

export default defineComponent({
  inheritAttrs: false,

  props: {
    options: {
      type: Object as PropType<google.maps.InfoWindowOptions>,
      default: () => ({}),
    },
  },

  emits: infoWindowEvents,

  setup(props, { slots, emit }) {
    const infoWindow = ref<google.maps.InfoWindow>();
    const infoWindowRef = ref<HTMLElement>();

    const map = inject(mapSymbol, ref());
    const api = inject(apiSymbol, ref());
    const anchor = inject(markerSymbol, ref());
    let anchorClickListener: google.maps.MapsEventListener;

    const hasSlotContent = computed(() => slots.default?.().some((vnode) => vnode.type !== Comment));

    onMounted(() => {
      watch(
        [map, () => props.options],
        ([_, options], [oldMap, oldOptions]) => {
          const hasChanged = !equal(options, oldOptions) || map.value !== oldMap;

          if (map.value && api.value && hasChanged) {
            if (infoWindow.value) {
              infoWindow.value.setOptions({
                ...options,
                content: hasSlotContent.value ? infoWindowRef.value : options.content,
              });

              if (!anchor.value) infoWindow.value.open({ map: map.value });
            } else {
              infoWindow.value = markRaw(
                new api.value.InfoWindow({
                  ...options,
                  content: hasSlotContent.value ? infoWindowRef.value : options.content,
                })
              );

              if (anchor.value) {
                anchorClickListener = anchor.value.addListener("click", () => {
                  if (!infoWindow.value) return;

                  infoWindow.value.open({
                    map: map.value,
                    anchor: anchor.value,
                  });
                });
              } else {
                infoWindow.value.open({ map: map.value });
              }

              infoWindowEvents.forEach((event) => {
                infoWindow.value?.addListener(event, (e: unknown) => emit(event, e));
              });
            }
          }
        },
        {
          immediate: true,
        }
      );
    });

    onBeforeUnmount(() => {
      if (anchorClickListener) anchorClickListener.remove();

      if (infoWindow.value) {
        api.value?.event.clearInstanceListeners(infoWindow.value);
        infoWindow.value.close();
      }
    });

    return { infoWindow, infoWindowRef, hasSlotContent };
  },
});
</script>

<style scoped>
.info-window-wrapper {
  display: none;
}

.mapdiv .info-window-wrapper {
  display: inline-block;
}
</style>
