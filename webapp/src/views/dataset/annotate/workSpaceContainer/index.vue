/** Copyright 2020 Tianshu AI Platform. All Rights Reserved.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 * =============================================================
 */

<template>
  <div class="workspace-container flex flex-col">
    <!-- zoom {{ zoom }}
    <div>brush: {{ brush }}</div>
    <div>api: {{ api }}</div>
    <div>annotation: {{ state.annotations }}</div>
    <div>tooltipData: {{ tooltipData }}</div>
    <div>imgInfo {{ imgInfo }}</div>
    <div>svgDimentsion, {{ svgDimension }}</div> -->
    <div v-if="!state.error.value" v-hotkey.stop="keymap" class="workspace-stage flex flex-col f1">
      <ToolBar
        :zoomIn="zoomIn"
        :zoomOut="zoomOut"
        :zoomReset="resetZoom"
        :clearSelection="clearSelection"
        :api="api"
        :setApi="setApi"
        :confirm="confirm"
        :isSegmentation="isSegmentation"
        v-on="listeners"
      />
      <div id="stage" ref="imgWrapperRef" v-loading="!imgInfo.loaded" class="f1 rel">
        <ZoomContainer
          ref="zoomRef"
          :controlled="true"
          :filter="filter"
          v-bind="zoom"
          :onZoom="handleZoom"
        >
          <div class="zoom-content">
            <div class="zoom-content-bound rel" :style="dimension.marginStyle">
              <div
                class="imgWrapper"
                :style="dimension.imgScaleStyle"
                :class="dimension.scale < 1 ? 'imgScale' : ''"
              >
                <img ref="imgRef" :src="currentImg.url" class="usn" />
              </div>
              <!-- svg ?????????????????????????????? -->
              <div class="annotation-element-group abs" :style="dimension.annotationGroupStyle">
                <svg ref="svgRef" class="canvas" :style="dimension.svg">
                  <g v-if="!isSegmentation">
                    <Brush
                      :stageWidth="dimension.svg.width"
                      :stageHeight="dimension.svg.height"
                      :onBrushStart="handleBrushStart"
                      :onBrushMove="handleBrushMove"
                      :onBrushEnd="handleBrushEnd"
                      :transformZoom="transformZoom"
                    />
                    <g class="annotation-group">
                      <BboxWrapper
                        v-for="annotate in api.annotations"
                        :key="annotate.id"
                        :annotate="annotate"
                        :brush="brush"
                        :offset="offset"
                        :transformer="transformer"
                        :svg="dimension.svg"
                        :scale="dimension.scale"
                        :bounds="dimension.img"
                        :onDragStart="onDragStart"
                        :onDragMove="onDragMove"
                        :onDragEnd="onDragEnd"
                        :onBrushHandleChange="onBrushHandleChange"
                        :onBrushHandleEnd="onBrushHandleEnd"
                        :currentAnnotationId="state.currentAnnotationId.value"
                        :setCurAnnotation="setCurAnnotation"
                        :getZoom="getZoom"
                      />
                    </g>
                  </g>
                  <Segmentation
                    v-else
                    ref="segmentationRef"
                    :stageWidth="dimension.svg.width"
                    :stageHeight="dimension.svg.height"
                    :shapes="api.shapes"
                    :offset="offsetBbox"
                    :currentAnnotationId="state.currentAnnotationId.value"
                    :setCurAnnotation="setCurAnnotation"
                    :updateState="updateState"
                    :transformZoom="transformZoom"
                    :scale="dimension.scale"
                    :getZoom="getZoom"
                    :bounds="dimension.img"
                    @change="handleShapesChange"
                  />
                </svg>
                <div class="annotation-extra">
                  <div
                    v-if="state.showScore.value && !isSegmentation"
                    class="annotation-score-group"
                  >
                    <Score
                      v-for="annotate in api.annotations"
                      :key="annotate.id"
                      :annotate="annotate"
                      :currentAnnotationId="state.currentAnnotationId.value"
                      :brush="brush"
                      :offset="offset"
                      :transformer="transformer"
                    />
                  </div>
                  <div v-if="state.showTag.value" class="annotation-tag-group">
                    <Tag
                      v-for="annotate in annotations"
                      :key="annotate.id"
                      :annotate="annotate"
                      :currentAnnotationId="state.currentAnnotationId.value"
                      :isMoving="isSegmentation ? transformer.isDragging : brush.isBrushing"
                      :offset="offset"
                      :transformer="transformer"
                      :getLabelName="getLabelName"
                      :annotationType="annotationType"
                    />
                  </div>
                  <div v-if="state.showId.value && isTrack" class="annotation-tag-group">
                    <AnnotationId
                      v-for="annotate in api.annotations"
                      :key="annotate.id"
                      :annotate="annotate"
                      :currentAnnotationId="state.currentAnnotationId.value"
                      :brush="brush"
                      :offset="offset"
                      :transformer="transformer"
                      :scale="dimension.scale"
                      :getLabelName="getLabelName"
                      :imgBounding="api.imgBounding"
                    />
                  </div>
                  <!-- ?????????????????????????????? -->
                  <BrushTip
                    v-if="brush.isBrushing && brush.extent"
                    :brush="brush"
                    :dimension="dimension"
                  />
                </div>
              </div>
            </div>
          </div>
        </ZoomContainer>
        <DropDownLabel
          :visible="tooltipData.visible"
          v-bind="tooltipData"
          :hideTooltip="hideTooltip"
          :value="api.label.value"
          :handleChange="handleSelectChange"
          :labels="labels"
        />
      </div>
    </div>
    <Exception v-else>
      <template slot="desc">
        {{ (state.error.value || {}).message || 'error' }}
      </template>
    </Exception>
  </div>
</template>
`
<script>
import {
  ref,
  computed,
  watch,
  reactive,
  inject,
  onMounted,
  onBeforeUnmount,
} from '@vue/composition-api';
import { isNil } from 'lodash';
import { event as d3Event } from 'd3-selection';
import { Message } from 'element-ui';

import { labelsSymbol } from '@/views/dataset/util';
import { useBrush, useZoom, unref, useTooltip, useImage } from '@/hooks';
import {
  getBounding,
  raise,
  replace,
  extent2Bbox,
  parsePolygon2Bbox,
  getZoomPosition,
} from '@/utils';
import { Brush } from '@/components/svg';
import ZoomContainer from '@/components/ZoomContainer';
import Exception from '@/components/Exception';
import ToolBar from './toolbar';
import BboxWrapper from './bboxWrapper';
import Score from './score';
import Tag from './tag';
import AnnotationId from './annotationId';
import DropDownLabel from './dropdownLabel';
import BrushTip from './brushTip';
import Segmentation from './segmentation';

const addEventListener = require('add-dom-event-listener');

const FooterHeight = 0;

// ???????????????
export const ThumbWidth = 160;

// msg ??????
let msgInstance = null;

export default {
  name: 'WorkSpaceContainer',
  components: {
    ZoomContainer,
    Segmentation,
    Exception,
    Brush,
    ToolBar,
    DropDownLabel,
    Score,
    Tag,
    AnnotationId,
    BboxWrapper,
    BrushTip,
  },
  props: {
    state: Object,
    currentImg: {
      type: Object,
      default: () => null,
    },
    drawShapeEnd: Function,
    createLabel: Function,
    queryLabels: Function,
    getLabelName: Function,
    updateState: Function,
    handleConfirm: Function,
    deleteAnnotation: Function,
    isTrack: Boolean,
    isSegmentation: Boolean,
    annotationType: String,
  },
  setup(props, ctx) {
    const {
      drawShapeEnd,
      state,
      createLabel,
      queryLabels,
      updateState,
      deleteAnnotation,
      handleConfirm,
      isSegmentation,
      annotationType,
    } = props;

    const imgWrapperRef = ref(null);
    const svgRef = ref(null);
    const resizerRef = ref(null); // ????????????
    const imgRef = ref(null); // ??????
    const segmentationRef = ref(null);

    // ????????????????????????
    const labels = inject(labelsSymbol);
    // ???????????????
    const api = reactive({
      annotations: props.state.annotations,
      shapes: props.state.shapes,
      label: {}, // ??????????????????????????????????????????
      bounding: null, // ??????????????????
      isCenter: false, // ?????????????????????
      imgBounding: null, // ????????????????????? bbox ??????????????????
      active: '', // ????????????
    });

    // ?????????????????????
    const annotations = computed(() => api[annotationType]);

    // ????????????
    const transformer = reactive({
      id: undefined,
      dx: 0,
      dy: 0,
      x: undefined,
      y: undefined,
    });

    const { listeners } = ctx;
    const {
      brush,
      onBrushStart,
      onBrushMove,
      updateBrush,
      getExtent,
      // onBrushEnd,
      onBrushReset,
    } = useBrush();

    const initialZoom = {
      zoom: unref(state.zoom),
      zoomX: unref(state.zoomX),
      zoomY: unref(state.zoomY),
    };

    // ???????????????????????????
    const { zoomIn, zoomOut, setZoom, reset: resetZoom, zoom, getZoom } = useZoom(
      initialZoom,
      imgWrapperRef
    );

    // tooltip
    const { tooltipData, showTooltip, hideTooltip } = useTooltip();

    // ????????????
    const { imgInfo = {}, setImg } = useImage();

    const filter = () => {
      // ????????????????????????
      // if (!state.selection.value) return true
      // ??????????????????????????????????????????
      return d3Event.type !== 'mousedown';
    };

    const setApi = (params) => {
      Object.assign(api, params);
    };

    // ??????????????????
    const setTransformer = (params) => {
      Object.assign(transformer, params);
    };

    // ?????? zoom ??????
    const transformZoom = (point) => {
      return getZoomPosition(ctx.refs.zoomRef.wrapperRef, point);
    };

    const defaultDimension = {
      svg: {},
      img: {},
      wrapper: {},
      margin: {},
      scale: 1,
    };

    // ?????????????????????????????? svg ????????????
    const dimension = computed(() => {
      // ??? dom ??????????????????????????? svg ??????
      if (!isNil(api.bounding) && !!imgInfo.loaded && svgRef.value) {
        // ??????????????????????????????
        const { width: cw, height: ch, left, top } = api.bounding;
        // ????????????????????????
        const iw = imgInfo.width;
        const ih = imgInfo.height;

        const wrapperDimension = {
          width: cw,
          height: ch - FooterHeight,
          left,
          top,
        };

        // ????????????????????????????????????????????????
        const imgScale = Math.min(
          wrapperDimension.width / imgInfo.width,
          wrapperDimension.height / imgInfo.height,
          1
        );

        // ???????????????????????????????????????????????????
        const svgDimension = {
          width: imgScale < 1 ? cw : Math.min(iw, cw),
          height: imgScale < 1 ? ch - FooterHeight : Math.min(ih, ch),
        };

        // ???????????????????????????
        const annotationGroupStyle = {
          left: imgScale === 1 ? `${(cw - iw) / 2}px` : 0,
          top: imgScale === 1 ? `${(ch - FooterHeight - ih) / 2}px` : 0,
          width: imgScale === 1 ? `${iw}px` : `${cw}px`,
          height: imgScale === 1 ? `${ih}px` : `${ch - FooterHeight}px`,
        };

        // ??????????????????margin: 0 auto ??????????????????
        const ml = Math.max(wrapperDimension.width - iw * imgScale, 0) / 2;
        const mt = Math.max(wrapperDimension.height - ih * imgScale, 0) / 2;

        const marginStyle = {};
        if (ml > 0) {
          marginStyle['padding-left'] = `${ml}px`;
          marginStyle['padding-right'] = `${ml}px`;
        }
        if (mt > 0) {
          marginStyle['padding-top'] = `${mt}px`;
          marginStyle['padding-bottom'] = `${mt}px`;
        }
        const margin = {
          left: ml,
          top: mt,
        };

        const imgDimension = {
          width: iw,
          height: ih,
        };

        const imgScaleStyle =
          imgScale < 1
            ? {
                width: `${imgInfo.width * imgScale}px`,
                height: `${imgInfo.height * imgScale}px`,
                transition: 'opacity 0.4s',
                opacity: 1,
              }
            : {};
        // ??????
        api.isCenter = true;
        return {
          svg: svgDimension,
          img: imgDimension,
          wrapper: wrapperDimension,
          margin,
          marginStyle,
          scale: imgScale,
          imgScaleStyle,
          annotationGroupStyle,
        };
      }
      if (!imgInfo.loaded || (imgInfo.width === 0 && imgInfo.height === 0)) {
        // ???????????????????????????????????????
        return {
          ...defaultDimension,
          imgScaleStyle: {
            opacity: 0,
          },
        };
      }
      return defaultDimension;
    });

    const findImgIndex = () => {
      const { files, currentImgId } = state;
      const currentImgIndex = files.value.findIndex((d) => d.id === currentImgId.value);
      return { currentImgIndex, files };
    };

    const onMessageClose = () => {
      // ?????? message ??????
      msgInstance = null;
    };

    const handlePrev = () => {
      const { files, currentImgIndex } = findImgIndex();
      if (currentImgIndex > 0) {
        ctx.emit('changeImg', files.value[currentImgIndex - 1]);
      } else if (!msgInstance) {
        msgInstance = Message.warning({
          message: '?????????????????????????????????????????????',
          onClose: onMessageClose,
        });
      }
    };

    const handleNext = () => {
      // ?????????????????????????????????????????????????????????
      if (props.state.filterUnfinished.value) return;

      const { files, currentImgIndex } = findImgIndex();
      if (currentImgIndex > -1 && currentImgIndex < files.value.length - 1) {
        ctx.emit('changeImg', files.value[currentImgIndex + 1]);
        // ?????????????????????
        ctx.emit('nextPage', files.value[currentImgIndex + 1], currentImgIndex + 1, files);
      } else if (!msgInstance) {
        msgInstance = Message.warning({
          message: '?????????????????????????????????????????????',
          onClose: onMessageClose,
        });
      }
    };

    // ????????????
    const removeAnnotation = () => {
      hideTooltip();
      const currentAnnotationId = state.currentAnnotationId.value;
      if (currentAnnotationId) {
        deleteAnnotation(currentAnnotationId);
      }
    };

    // ?????????????????????????????????????????????
    const cancelSelection = () => {
      segmentationRef.value?.reset();
    };

    // ?????????????????????????????????????????????
    const finishDraw = () => {
      segmentationRef.value?.finishDraw();
    };

    // ???????????????????????????
    const selection = () => {
      setApi({ active: 'selection' });
      // ??????????????? dropdown
      hideTooltip();
      listeners.selection(true);
    };

    watch(
      () => api.isCenter,
      (isCenter) => {
        if (isCenter) {
          const { width: boundingWidth, height: boundingHeight } = api.bounding;
          const { width: imgWidth, height: imgHeight } = getBounding(imgRef.value);
          // ???????????????????????????????????????????????????
          const mw = dimension.value.scale < 1 ? boundingWidth : dimension.value.img.width;
          const mh =
            dimension.value.scale < 1 ? boundingHeight - FooterHeight : dimension.value.img.height;
          Object.assign(api, {
            imgBounding: [(mw - imgWidth) / 2, (mh - imgHeight) / 2],
          });
        }
      }
    );

    // ?????????
    const keymap = computed(() => ({
      left: {
        keyup: handlePrev,
      },
      right: {
        keyup: handleNext,
      },
      backspace: removeAnnotation,
      delete: removeAnnotation,
      n: selection,
      esc: cancelSelection,
      f: finishDraw,
    }));

    // ????????????
    const setCurAnnotation = (annotation = {}) => {
      updateState({
        currentAnnotationId: annotation.id || '',
      });
    };

    // ????????????
    const handleBrushStart = (start) => {
      // ??????????????? dropdown
      hideTooltip();
      // ????????????????????????
      // if (!state.selection.value) return;
      const { x, y } = start;
      onBrushStart({ x, y });
      // ???????????????????????????
      setCurAnnotation(undefined);
    };

    const handleBrushMove = (state) => {
      const { x, y } = state.end || {};
      onBrushMove({ x, y });
    };

    const handleBrushEnd = (state, event, options = {}) => {
      const { prevState = {} } = options;
      // ?????????move ????????????
      if (state.end && !!prevState.isDragging) {
        // ??????tooltip
        showTooltip({}, event, { el: imgWrapperRef.value });
        // ??????
        drawShapeEnd && drawShapeEnd(state, event);
        onBrushReset();
        return;
      }
      onBrushReset();
    };

    // ???????????????
    const clearSelection = () => listeners.selection(false);

    // ???????????????????????????
    const islabelExists = (value) => {
      return !!(labels.value || []).find((label) => label.id === Number(value));
    };

    // ?????????????????? bbox
    const offsetBbox = (bbox) => {
      const paddingLeft =
        dimension.value.scale < 1 && !isNil(api.imgBounding) ? api.imgBounding[0] : 0;

      const paddingTop =
        dimension.value.scale < 1 && !isNil(api.imgBounding) ? api.imgBounding[1] : 0;

      const pos = {
        x: bbox.x * dimension.value.scale + paddingLeft,
        y: bbox.y * dimension.value.scale + paddingTop,
      };

      if (bbox.width && bbox.height) {
        Object.assign(pos, {
          width: bbox.width * dimension.value.scale,
          height: bbox.height * dimension.value.scale,
        });
      }

      return pos;
    };

    // ????????????
    const offset = (annotate) => {
      const { data = {} } = annotate;
      const _bbox = isSegmentation ? parsePolygon2Bbox(data.points) : extent2Bbox(data.extent);
      return offsetBbox(_bbox);
    };

    // handle ??????
    const onBrushHandleChange = (brush, annotation) => {
      // ?????? brush
      const pos = offset(annotation);
      updateState((prev) => {
        const index = prev.annotations.findIndex((d) => d.id === annotation.id);
        if (index > -1) {
          const selectedItem = prev.annotations[index];
          const _nextItem = {
            ...selectedItem,
            data: {
              ...selectedItem.data,
              extent: brush.extent,
            },
          };

          const nextAnnotations = replace(prev.annotations, index, _nextItem);
          return {
            ...prev,
            annotations: nextAnnotations,
          };
        }
        return prev;
      });

      // ??????brush
      updateBrush((prevBrush) => {
        return {
          ...prevBrush,
          isBrushing: true,
          extent: {
            x0: pos.x,
            x1: pos.x + pos.width,
            y0: pos.y,
            y1: pos.y + pos.height,
          },
        };
      });
    };

    // handle ????????????
    const onBrushHandleEnd = (brush, annotation) => {
      // ?????? brush
      const pos = offset(annotation);
      // ??????brush
      updateBrush((prevBrush) => {
        return {
          ...prevBrush,
          isBrushing: false,
          extent: {
            x0: pos.x,
            x1: pos.x + pos.width,
            y0: pos.y,
            y1: pos.y + pos.height,
          },
        };
      });
    };

    // ????????????
    const confirm = () => {
      handleConfirm().then(() => {
        // ??????????????????
        Message.success({ message: '??????????????????', duration: 400, onClose: handleNext });
      });
    };

    // ??????selectItem
    const handleSelectChange = async (value) => {
      let labelVal = value;
      // ?????????????????????????????????
      // ???????????????????????????
      if (!islabelExists(value)) {
        labelVal = await createLabel({ name: value });
        const nextLabels = await queryLabels();
        // ???????????? provide
        updateState({
          labels: nextLabels,
        });
      }
      const { currentAnnotationId } = state;
      const annotationInfo = isSegmentation ? state.shapes : state.annotations;
      const selectedLabel = {
        ...api.label,
        value: labelVal,
      };
      Object.assign(api, {
        label: selectedLabel,
      });
      const curAnnotation =
        annotationInfo.value.find((d) => d.id === currentAnnotationId.value) || {};
      // ????????????????????????????????????
      ctx.emit('selectLabel', { selectedLabel, curAnnotation });
      // ?????????????????????????????????
      hideTooltip();
    };

    const handleZoom = (nextZoomTransform) => {
      if (!nextZoomTransform) return;
      setZoom({
        zoomX: nextZoomTransform.x,
        zoomY: nextZoomTransform.y,
        zoom: nextZoomTransform.k,
      });
    };

    // ??????????????????????????????
    const onDragStart = (draw, annotation) => {
      const index = api.annotations.findIndex((d) => d.id === annotation.id);
      if (index > -1) {
        const raised = raise(api.annotations, index);
        Object.assign(api, {
          annotations: raised,
        });
      }
      // ??????????????????
      setCurAnnotation(annotation);

      // ?????? brush
      const pos = offset(annotation);
      updateBrush((prevBrush) => {
        const start = {
          x: pos.x,
          y: pos.y,
        };
        const end = {
          x: pos.x + pos.width,
          y: pos.y + pos.height,
        };
        return {
          ...prevBrush,
          start,
          end,
          extent: getExtent(start, end),
        };
      });
    };

    // ?????? boxing ????????????
    const onDragMove = (draw, annotation) => {
      const pos = offset(annotation);
      const { drag = {} } = draw;
      const { zoom } = getZoom();

      const validDx =
        drag.dx > 0
          ? Math.min(drag.dx / zoom, dimension.value.svg.width - pos.x - pos.width)
          : Math.max(drag.dx / zoom, -pos.x);

      const validDy =
        drag.dy > 0
          ? Math.min(drag.dy / zoom, dimension.value.svg.height - pos.y - pos.height)
          : Math.max(drag.dy / zoom, -pos.y);

      // ?????? brush ??????
      updateBrush((prevBrush) => {
        const { x: x0, y: y0 } = prevBrush.start;
        const { x: x1, y: y1 } = prevBrush.end;
        return {
          ...prevBrush,
          isBrushing: true,
          extent: {
            ...prevBrush.extent,
            x0: x0 + validDx,
            x1: x1 + validDx,
            y0: y0 + validDy,
            y1: y1 + validDy,
          },
        };
      });

      setTransformer({
        isDragging: true,
        id: annotation.id,
        x: drag.x,
        y: drag.y,
        dx: validDx,
        dy: validDy,
      });
    };

    // ?????? boxing ?????????????????????
    const onDragEnd = (draw, annotation) => {
      const { drag = {} } = draw;
      // ???????????? transform
      setTransformer({
        isDragging: false,
        id: annotation.id,
        x: drag.x,
        y: drag.y,
        dx: 0,
        dy: 0,
      });

      updateState((prev) => {
        const index = prev.annotations.findIndex((d) => d.id === annotation.id);
        if (index > -1) {
          const selectedItem = prev.annotations[index];
          const _nextItem = {
            ...selectedItem,
            data: {
              ...selectedItem.data,
              extent: {
                // ??????????????????????????????zoom
                x0: selectedItem.data.extent.x0 + (drag.validDx || 0),
                y0: selectedItem.data.extent.y0 + (drag.validDy || 0),
                x1: selectedItem.data.extent.x1 + (drag.validDx || 0),
                y1: selectedItem.data.extent.y1 + (drag.validDy || 0),
              },
            },
          };

          const nextAnnotations = replace(prev.annotations, index, _nextItem);
          return {
            ...prev,
            annotations: nextAnnotations,
          };
        }
        return prev;
      });

      // ?????? brush ??????
      updateBrush((prevBrush) => {
        return {
          ...prevBrush,
          isBrushing: false,
          start: {
            ...prevBrush.start,
            x: Math.min(prevBrush.extent.x0, prevBrush.extent.x1),
            y: Math.min(prevBrush.extent.y0, prevBrush.extent.y1),
          },
          end: {
            ...prevBrush.end,
            x: Math.max(prevBrush.extent.x0, prevBrush.extent.x1),
            y: Math.max(prevBrush.extent.y0, prevBrush.extent.y1),
          },
        };
      });
    };

    const handleShapesChange = ({ type, state: shapeState, options = {} }) => {
      const { shape, event } = options;
      switch (type) {
        case 'DRAW_END': {
          // ????????????????????? shape
          event &&
            showTooltip(shape, event, {
              el: imgWrapperRef.value,
            });
          // ?????????????????????????????????
          drawShapeEnd(shape);
          break;
        }
        default: {
          const { transformer } = segmentationRef.value;
          // ??????transformer
          setTransformer(transformer);
          updateState({
            shapes: shapeState.shapes,
          });
        }
      }
    };

    onMounted(() => {
      addEventListener(document.body, 'click', (e) => {
        // ????????????????????????????????????
        // ???????????????????????? table
        const { target } = e;
        if (
          !target.closest('.annotation-table') &&
          !target.closest('#stage') &&
          !target.closest('.label-list')
        ) {
          // ?????????????????????
          updateState({
            currentAnnotationId: '',
          });
        }
      });

      // resizerRef.value = addEventListener(window, 'resize', () => {
      //   api.bounding = getBounding(imgWrapperRef.value)
      // })
      // ?????????????????????
      api.bounding = getBounding(imgWrapperRef.value);
    });

    onBeforeUnmount(() => {
      resizerRef.value && resizerRef.value.remove();
      resizerRef.value = null;
    });

    // ?????? currentImage ??????
    watch(
      () => props.currentImg,
      (nextImg) => {
        // ???????????????????????? zoom
        resetZoom();
        // ??????????????????????????????
        Object.assign(api, {
          label: {},
          isCenter: false,
          imgBounding: null,
        });
        if (nextImg?.url) {
          setImg(nextImg.url);
        }
        // ????????????????????????
        setApi({
          [annotationType]: [],
        });
        // ????????????????????????
        cancelSelection();
      }
    );

    watch(
      () => props.state,
      (nextProps) => {
        if (annotationType in nextProps) {
          api[annotationType] = nextProps[annotationType] || [];
        }
      }
    );

    return {
      listeners,
      imgWrapperRef,
      // labels
      labels,
      brush,
      clearSelection,
      filter,
      // zoom
      zoom,
      zoomIn,
      zoomOut,
      resetZoom,
      handleZoom,
      // tooltip
      tooltipData,
      hideTooltip,
      // img
      imgInfo,
      dimension,
      svgRef,
      imgRef,
      // annotations
      api,
      setApi,
      // event
      handleSelectChange,
      confirm,
      onDragStart,
      onDragMove,
      onDragEnd,
      keymap,
      // brush ??????
      handleBrushStart,
      handleBrushMove,
      handleBrushEnd,
      // ????????????
      offset,
      // ??????????????????
      offsetBbox,
      transformer,
      setTransformer,
      onBrushHandleChange,
      onBrushHandleEnd,
      // ???????????????????????????????????????????????????
      transformZoom,
      getZoom,
      setCurAnnotation,
      segmentationRef,
      handleShapesChange,
      annotations,
    };
  },
};
</script>
<style lang="scss">
@import '~@/assets/styles/variables.scss';

#stage {
  max-height: 100%;
}

.workspace-stage {
  flex: 1;
  overflow: hidden;

  .imgWrapper {
    &.imgScale {
      margin: 0 auto;

      img {
        display: inline-block;
        width: 100%;
        height: 100%;
        user-select: none;
      }
    }
  }

  .canvas {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: calc(100vh - 50px - 48px);
  }

  .annotation-score-group {
    pointer-events: none;

    .annotation-score-row {
      position: absolute;
      color: #fff;

      .score {
        display: inline-block;
        min-width: 48px;
        font-size: 16px;
        line-height: 24px;
        user-select: none;

        .unit {
          margin-left: 2px;
          font-size: 0.8em;
        }
      }
    }
  }

  .annotation-tag-group {
    pointer-events: none;

    .annotation-label {
      position: absolute;
      color: #fff;
    }
  }

  .bbox-group {
    cursor: pointer;
  }

  .brush-tooltip {
    position: absolute;
    padding: 7px 12px;
    font-size: 12px;
    line-height: 1em;
    color: #fff;
    pointer-events: none;
    background-color: $dark;
    border-radius: 4px;

    .tooltip-item-row {
      display: flex;
      white-space: nowrap;
    }
  }
}
</style>
