<template>
	<div
		:ref="setElements(object, index)"
		@mousedown.left="handleMouseDown($event, object)"
		@mouseup="handleMouseUp($event, width)"
		@mousemove="mouseMoveHandler($event, width)"
		@mouseleave="mouseLeaveHandler(width)"
		:style="[
			postionalStyles(startX, startY, width, height),
			style.zIndex && { zIndex: style.zIndex },
		]"
		:class="[
			'table-container',
			classes,
			removeOuterBorder === 'YesKeepBottom' && 'no-outer-border-keep-bottom',
			removeOuterBorder && removeOuterBorder !== 'YesKeepBottom' && 'no-outer-border',
			MainStore.getCurrentElementsId.includes(id) && 'active-elements',
		]"
	>
		<div
			:style="['overflow: hidden;', widthHeightStyle(width, height)]"
			@click.self="
				() => {
					selectedColumn = null;
					selectedDynamicText = null;
				}
			"
		>
            <table class="printTable">
                <thead>
                    <tr v-if="columns.length">
                        <th
                            :style="[
                                'cursor: pointer;',
                                column.width && {
                                    width: `${column.width}%`,
                                    maxWidth: `${column.width}%`,
                                },
                                headerStyleSansRadius,
                                headerCornerStyle(index),
                                removeOuterBorder && headerOuterEdgeStyle(index),
                                column.applyStyleToHeader && column.style,
                            ]"
                            v-for="(column, index) in columns"
                            :class="
                                (menu?.index == index || column == selectedColumn) &&
                                'current-column'
                            "
							:key="column.fieldname"
							:draggable="columnDragging"
							@dragstart="dragstart($event, index)"
							@drop="drop($event, index)"
							@dragleave="dragleave"
							@dragover="allowDrop"
							@contextmenu.prevent="handleMenu($event, index)"
							@mousedown="handleColumnClick(column)"
							@dblclick.stop="handleDblClick(table, column)"
							:ref="
								(el) => {
									column.DOMRef = el;
								}
							"
						>
							<span :class="{ emptyColumnHead: !Boolean(column.label.length) }">{{
								column?.label
							}}</span>
							<div
								class="resizer"
								draggable="false"
								:ref="
									(el) => {
										column.DOMRef = el;
									}
								"
								@mousedown.left="mouseDownHandler($event, column)"
								@mouseup.left="mouseUpHandler"
								:style="{
									height: DOMRef
										? DOMRef.firstChild.getBoundingClientRect().height + 'px'
										: '100%',
								}"
							></div>
						</th>
					</tr>
				</thead>
                <tbody>
                    <tr
                        v-if="columns.length"
                        v-for="(row, rowIdx) in previewRows"
                        :key="row.idx ?? rowIdx"
                    >
                        <BaseTableTd
                            v-bind="{
                                row,
                                columns,
                                style,
                                labelStyle,
                                selectedDynamicText,
                                setSelectedDynamicText,
                                table,
                                altStyle,
                                removeOuterBorder,
                                isLastRow: rowIdx === previewRows.length - 1,
                            }"
                        />
                    </tr>
                </tbody>
            </table>
		</div>
		<BaseResizeHandles
			v-if="
				MainStore.activeControl == 'mouse-pointer' &&
				MainStore.getCurrentElementsId.includes(id)
			"
		/>
		<AppTableContextMenu v-if="menu" v-bind="{ menu }" @handleMenuClick="handleMenuClick" />
	</div>
</template>

<script setup>
import { useMainStore } from "../../store/MainStore";
import { useElement } from "../../composables/Element";
import {
	postionalStyles,
	setCurrentElement,
	lockAxis,
	cloneElement,
	deleteCurrentElements,
	widthHeightStyle,
} from "../../utils";
import { toRefs, ref, watch, computed } from "vue";
import { useDraw } from "../../composables/Draw";
import BaseResizeHandles from "./BaseResizeHandles.vue";
import AppTableContextMenu from "../layout/AppTableContextMenu.vue";
import BaseTableTd from "./BaseTableTd.vue";
import { onClickOutside } from "@vueuse/core";
const MainStore = useMainStore();
const props = defineProps({
	object: {
		type: Object,
		required: true,
	},
	index: {
		type: Number,
		required: true,
	},
});
const isResizeOn = ref(false);
const currentColumn = ref(null);
const columnDragging = ref(true);
const draggableEl = ref(-1);
const menu = ref(null);

const {
	id,
	table,
	columns,
	startX,
	startY,
	width,
	height,
	style,
	labelStyle,
	headerStyle,
	altStyle,
	removeOuterBorder,
	classes,
	PreviewRowNo,
	styleEditMode,
	selectedColumn,
	selectedDynamicText,
	DOMRef,
} = toRefs(props.object);

// Prevent header border-radius from applying to every cell; only round outer corners
const headerStyleSansRadius = computed(() => {
    const s = { ...(headerStyle?.value || {}) };
    if (s.borderRadius !== undefined) delete s.borderRadius;
    return s;
});

const headerCornerStyle = (idx) => {
    const radius = headerStyle?.value?.borderRadius || 0;
    if (!radius) return {};
    const lastIdx = (columns?.value?.length || 0) - 1;
    return {
        borderRadius: 0,
        ...(idx === 0
            ? { borderTopLeftRadius: radius, borderBottomLeftRadius: radius }
            : {}),
        ...(idx === lastIdx
            ? { borderTopRightRadius: radius, borderBottomRightRadius: radius }
            : {}),
    };
};

// Inline suppression for header outer edges when removeOuterBorder is enabled
const headerOuterEdgeStyle = (idx) => {
    if (!removeOuterBorder?.value) return "";
    const lastIdx = (columns?.value?.length || 0) - 1;
    const styles = [];
    styles.push("border-top-style: none !important;");
    if (idx === 0) styles.push("border-left-style: none !important;");
    if (idx === lastIdx) styles.push("border-right-style: none !important;");
    return styles.join(" ");
};

watch(
    () => selectedColumn.value,
    (value) => {
        if (value) {
            MainStore.frappeControls.applyStyleToHeader?.set_value(value.applyStyleToHeader);
        }
    }
);

// Rows visible in preview; used to decide last row for bottom-edge logic
const previewRows = computed(() => {
    const rows = MainStore.docData[table?.value?.fieldname] || [];
    const count = PreviewRowNo?.value || 3;
    const sliced = rows.slice(0, count);
    return sliced.length ? sliced : [{}];
});

const setSelectedDynamicText = (value, isLabel) => {
	if (
		selectedDynamicText.value === value &&
		styleEditMode.value == (isLabel ? "label" : "main")
	) {
		selectedDynamicText.value = null;
	} else {
		selectedDynamicText.value = value;
		selectedColumn.value = null;
		let removeSelectedText = onClickOutside(DOMRef.value, (event) => {
			for (let index = 0; index < event.composedPath().length; index++) {
				if (event.composedPath()[index].id === "canvas") {
					selectedDynamicText.value = null;
					removeSelectedText();
				}
			}
		});
	}
	styleEditMode.value = isLabel ? "label" : "main";
};

const handleColumnClick = (column) => {
	if (!props.object.table) {
		MainStore.frappeControls.table.set_focus();
	} else if (MainStore.activeControl == "mouse-pointer") {
		selectedColumn.value = column;
		MainStore.frappeControls.tableStyleEditMode?.set_value("main");
		selectedDynamicText.value = null;
		let removeSelectedColumn = onClickOutside(DOMRef.value, (event) => {
			for (let index = 0; index < event.composedPath().length; index++) {
				if (event.composedPath()[index].id === "canvas") {
					selectedColumn.value = null;
					removeSelectedColumn();
				}
			}
		});
	}
};

const handleDblClick = (table, column) => {
	if (!props.object.table) {
		MainStore.frappeControls.table.set_focus();
		MainStore.frappeControls.table.$input.one("blur", () => {
			MainStore.openTableColumnModal = {
				table: MainStore.getCurrentElementsValues[0]?.table,
				column,
			};
		});
	} else {
		MainStore.openTableColumnModal = { table, column };
	}
};

const handleMenuClick = (index, action) => {
	switch (action) {
		case "before":
			columns.value.splice(index, 0, {
				id: index,
				label: "",
				style: {},
				applyStyleToHeader: false,
			});
			break;
		case "after":
			columns.value.splice(index + 1, 0, {
				id: index + 1,
				label: "",
				style: {},
				applyStyleToHeader: false,
			});
			break;
		case "delete":
			columns.value.splice(index, 1)[0].dynamicContent?.forEach((el) => {
				MainStore.dynamicData.splice(MainStore.dynamicData.indexOf(el), 1);
			});
			break;
	}
	columns.value.forEach((element, index) => {
		element.id = index;
	});
	menu.value.index = null;
};

const { setElements } = useElement({
	draggable: true,
	resizable: true,
});

const handleMenu = (e, index) => {
	if (!menu.value) {
		menu.value = {
			left: e.x - DOMRef.value.getBoundingClientRect().x + "px",
			top: e.y - DOMRef.value.getBoundingClientRect().y + "px",
			index: index,
		};
	} else {
		menu.value.left = e.x - DOMRef.value.getBoundingClientRect().x + "px";
		menu.value.top = e.y - DOMRef.value.getBoundingClientRect().y + "px";
		menu.value.index = index;
	}
};

const dragstart = (ev, index) => {
	draggableEl.value = index;
	ev.dataTransfer.dropEffect = "move";
};
const dragleave = (ev) => {
	ev.target.classList.remove("dropzone");
};
const allowDrop = (ev) => {
	ev.dataTransfer.dropEffect = "move";
	ev.target.classList.add("dropzone");
	ev.preventDefault();
};
const drop = (ev, index) => {
	ev.target.classList.remove("dropzone");
	columns.value.splice(index, 0, columns.value.splice(draggableEl.value, 1)[0]);
	columns.value.forEach((element, index) => {
		element.id = index;
	});
	ev.preventDefault();
	draggableEl.value = null;
};
const mouseDownHandler = (e, column) => {
	column.x = e.clientX;
	columnDragging.value = false;
	isResizeOn.value = true;
	column.DOMRef = e.target;
	!currentColumn.value && (currentColumn.value = column);
	column.DOMRef.classList.add("resizer-active");
	column.w = e.target.parentElement.getBoundingClientRect().width;
	DOMRef.value.style.cursor = "col-resize";
};
const mouseLeaveHandler = (tablewidth) => {
	if (isResizeOn.value) {
		columns.value.forEach((column) => {
			column.width = (column.DOMRef.getBoundingClientRect().width * 100) / tablewidth;
		});
	}
	if (currentColumn.value) {
		document.querySelector(".resizer-active").classList.remove("resizer-active");
		columnDragging.value = true;
		currentColumn.value = null;
		isResizeOn.value = false;
		DOMRef.value.style.cursor = "";
	}
};

const mouseMoveHandler = (e, tablewidth) => {
	if (!isResizeOn.value || !currentColumn.value) return;
	const dx = e.clientX - currentColumn.value.x;
	const newWidth = dx + currentColumn.value.w;
	currentColumn.value.width = (newWidth * 100) / tablewidth;
	if (currentColumn.value.width < 2) {
		currentColumn.value.width = 2;
	}
};
const { drawEventHandler, parameters } = useDraw();

const handleMouseDown = (e, element = null) => {
	MainStore.setActiveControl("MousePointer");
	lockAxis(element, e.shiftKey);
	e.stopPropagation();
	MainStore.isMoveStart = true;
	if (e.altKey) {
		element && setCurrentElement(e, element);
		cloneElement();
	} else {
		element && setCurrentElement(e, element);
	}
	drawEventHandler.mousedown(e);
	MainStore.currentDrawListener = { drawEventHandler, parameters };
	e.stopPropagation();
};

const handleMouseUp = (e, tablewidth) => {
	if (currentColumn.value) {
		document.querySelector(".resizer-active").classList.remove("resizer-active");
		currentColumn.value = null;
		isResizeOn.value = false;
		DOMRef.value.style.cursor = "";
	}
	columnDragging.value = true;
	if (MainStore.lastCloned && !MainStore.isMoved && MainStore.activeControl == "mouse-pointer") {
		deleteCurrentElements();
	}
	MainStore.currentDrawListener?.drawEventHandler.mouseup(e);
	MainStore.setActiveControl("MousePointer");
	MainStore.isMoved = MainStore.isMoveStart = false;
	MainStore.lastCloned = null;
	if (MainStore.frappeControls.table?.get_value() == "") {
		MainStore.frappeControls.table.set_focus();
	}
};
</script>

<style lang="scss" scoped>
.emptyColumnHead::after {
	content: "Select Field";
}
.dropzone {
	background-color: var(--gray-300);
}
.table-container {
	background-color: var(--gray-50);
	.printTable {
		border-collapse: collapse;
		box-sizing: border-box;
		border-spacing: 0px;
		width: 100%;
		overflow: hidden;

		tr:first-child th {
			border-top-style: solid !important;
		}
		tr th:first-child {
			border-left-style: solid !important;
		}
		tr th:last-child {
			border-right-style: solid !important;
		}
		.current-column {
			outline: 1.5px solid var(--primary-color);
			outline-offset: -1.5px;
		}
	}
	th {
		position: relative;
	}
	.resizer {
		position: absolute;
		top: 0;
		right: 0;
		width: 5px;
		cursor: col-resize;
		user-select: none;
		border-right: 1px solid transparent;
	}
	&.no-outer-border {
		.printTable {
			tr:first-child th {
				border-top-style: none !important;
			}
			tr th:first-child {
				border-left-style: none !important;
			}
			tr th:last-child {
				border-right-style: none !important;
			}
			/* Remove outer borders on body cells via deep selectors (child component) */
			:deep(.printTable tr td:first-child) {
				border-left-style: none !important;
			}
			:deep(.printTable tr td:last-child) {
				border-right-style: none !important;
			}
			/* Also remove bottom border on the last row via deep selector */
			:deep(.printTable tr:last-child td) {
				border-bottom-style: none !important;
			}
		}
	}

	&.no-outer-border-keep-bottom {
		.printTable {
			tr:first-child th {
				border-top-style: none !important;
			}
			tr th:first-child {
				border-left-style: none !important;
			}
			tr th:last-child {
				border-right-style: none !important;
			}
			/* Remove outer left/right on body cells but keep bottom */
			:deep(.printTable tr td:first-child) {
				border-left-style: none !important;
			}
			:deep(.printTable tr td:last-child) {
				border-right-style: none !important;
			}
		}
	}
}

.resizer:hover,
.resizer-active,
.resizing {
	border-right: 2px solid var(--primary-color);
}

[contenteditable] {
	outline: none;
}
[contenteditable]:empty:before {
	content: attr(data-placeholder);
}
[contenteditable]:empty:focus:before {
	content: "";
}
.text-hover:hover {
	box-sizing: border-box !important;
	border-bottom: 1px solid var(--primary-color) !important;
}
</style>
