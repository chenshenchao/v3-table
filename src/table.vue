<template>
    <div ref="rootElement" class="v3-table" :class="styleClasses">
        <div
            ref="foreElement"
            class="v3-table-fore"
            :style="foreStyle"
            @dragover="onHeadCellDragOver"
            @drop="onHeadCellDrop"
            @mousewheel="onMouseWheel"
        >
            <table v-if="'default' in slots" class="v3-table-main" :style="tableStyle">
                <tr ref="headElement" class="v3-table-head">
                    <th
                        class="v3-table-head-cell"
                        v-for="(column, i) in columns"
                        :key="i"
                        :style="headStyles[i]"
                    >
                        <v3-table-head-cell
                            :renderer="column.children?.head"
                            :content="column.props"
                        />
                        <div
                            class="v3-table-head-cell-drag"
                            draggable="true"
                            @dragstart="onHeadCellDragStart"
                            @dragend="onHeadCellDragEnd"
                        ></div>
                    </th>
                </tr>
                <tr class="v3-table-row" v-for="(row, i) in rows" :key="i" :style="rowStyle">
                    <td class="v3-table-row-cell" v-for="(column, j) in columns" :key="j">
                        <v3-table-row-cell
                            :renderer="column.children?.default"
                            :content="{ row: row, index: start + i }"
                        />
                    </td>
                </tr>
            </table>
        </div>
        <div ref="backElement" class="v3-table-back" :style="backStyle" @scroll="onVScroll">
            <div ref="fillElement" class="v3-table-fill" :style="fillStyle"></div>
        </div>
        <slot></slot>
    </div>
</template>

<script setup>
import { V3TableRowCell, V3TableHeadCell } from "./cell.js";
import {
    onMounted,
    useSlots,
    ref,
    onBeforeUnmount,
    computed,
    reactive,
} from "vue";
import { throttle } from "lodash";

const rootElement = ref();
const foreElement = ref();
const backElement = ref();
const headElement = ref();
const fillElement = ref();

const props = defineProps({
    rows: {
        type: Array,
        required: true,
        default: () => [],
    },
    height: {
        type: [Number, null],
        required: false,
        default: null,
    },
    rowHeight: {
        type: Number,
        required: false,
        default: 30,
    },
    columnResizable: {
        type: Boolean,
        required: false,
        default: false,
    },
});

const slots = useSlots();
const start = ref(0);
const rowCount = ref(0);
const dragging = ref(false);
const sdw = ref(0);

const columns = computed(() => {
    return slots.default();
});

const rows = computed(() => {
    return props.rows.slice(start.value, start.value + rowCount.value);
});

const columnWidths = reactive([]);

for (let column of columns.value) {
    columnWidths.push(column.props.width ?? 100);
}

const backStyle = reactive({
    top: 0,
});
const foreStyle = reactive({
    width: "auto",
});

const fillStyle = reactive({
    height: "100%",
});
const rowStyle = reactive({
    height: `${props.rowHeight}px`,
});

const styleClasses = computed(() => {
    const result = [];
    if (props.height == null) {
        result.push("filling");
    }
    return result;
});

const tableStyle = computed(() => {
    return {
        // width: `${tableWidth.value}px`
    };
});

const headStyles = computed(() => {
    const result = [];
    for (let i in columns.value) {
        const style = {};
        style.minWidth = `${columnWidths[i]}px`;
        result.push(style);
    }
    return result;
});

const onHeadCellDragStart = e => {
    sdw.value = e.screenX;
    console.log('drag start', e);
    dragging.value = true;
};

const onHeadCellDrag = e => {
    console.log('drag', e);
};

const onHeadCellDragEnd = e => {
    const nsdw = e.screenX;
    console.log('drag end',  nsdw - sdw.value + foreElement.value.scrollLeft);
    dragging.value = false;
};

const onHeadCellDragOver = e => {
    e.preventDefault();
    // console.log('drag over', e);
};

const onHeadCellDrop = e => {
    console.log('drop', e);
};

const onMouseWheel = e => {
    backElement.value.scrollBy(0, -e.wheelDelta);
}

const onResize = throttle(() => {
    const bcw = backElement.value.clientWidth;
    const bch = backElement.value.clientHeight;

    rowCount.value = Math.ceil(bch / props.rowHeight);

    foreStyle.width = `${bcw}px`;
    backStyle.top = `${headElement.value.offsetHeight}px`;
    backStyle.bottom = `${foreElement.value.offsetHeight - foreElement.value.clientHeight
        }px`;
    fillStyle.height = `${props.rowHeight * props.rows.length}px`;
}, 600);

const onVScroll = throttle((e) => {
    const bst = backElement.value.scrollTop;
    const bse = fillElement.value.offsetHeight - props.rowHeight * 2;
    start.value = Math.floor((bst / bse) * props.rows.length);
}, 100);

const observer = new ResizeObserver(onResize);

onMounted(() => {
    onResize();
    observer.observe(rootElement.value);
});

onBeforeUnmount(() => {
    observer.disconnect();
});
</script>

<style lang="scss" scoped>
.v3-table {
    position: relative;
    overflow: hidden;
    border: 1px solid #f6f6f6;
    box-sizing: border-box;
    background: #fff;

    &.filling {
        width: 100%;
        height: 100%;
    }
}

.v3-table-fore {
    display: block;
    overflow-x: auto;
    overflow-y: hidden;
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    z-index: 10;
    width: 100%;
    background: #f6f6f6f6;
}

.v3-table-drag-pane {
    display: block;
    overflow: hidden;
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    z-index: 100;
    width: 100%;
    background: #2222;
}

.v3-table-head-cell {
    position: relative;
    box-sizing: border-box;
    padding: 3px 5px;
}

.v3-table-head-cell-drag {
    position: absolute;
    top: 0;
    right: 0;
    bottom: 0;
    width: 5px;
    height: 100%;
    cursor: ew-resize;
}

.v3-table-back {
    display: block;
    overflow-x: hidden;
    overflow-y: auto;
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    z-index: 1;
    width: 100%;
    background: transparent;
}
</style>