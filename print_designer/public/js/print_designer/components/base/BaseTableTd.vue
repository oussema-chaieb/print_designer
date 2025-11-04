<template>
    <td
        v-for="(column, colIdx) in columns"
        :key="column.fieldname ?? colIdx"
        :style="[
            style,
            altStyle && row.idx % 2 == 0 && altStyle,
            column.style,
            removeOuterBorder && outerEdgeStyle(colIdx, columns.length, isLastRow),
        ]"
        @click.self="setSelectedDynamicText(null)"
    >
		<template
			v-for="field in column?.dynamicContent"
			:key="`${field?.parentField}${field?.fieldname}`"
		>
			<BaseDynamicTextSpanTag
				v-bind="{
					field,
					labelStyle,
					index: row.idx,
					selectedDynamicText,
					setSelectedDynamicText,
					parentClass: 'printTable',
					table,
				}"
			/>
		</template>
	</td>
</template>

<script setup>
import BaseDynamicTextSpanTag from "./BaseDynamicTextSpanTag.vue";
const props = defineProps({
	table: {
		type: Object,
		required: true,
	},
	row: {
		type: Object,
		required: true,
	},
	columns: {
		type: Object,
		required: true,
	},
	style: {
		type: Object,
		required: true,
	},
	labelStyle: {
		type: Object,
		required: true,
	},
	altStyle: {
		type: Object,
		required: false,
	},
	selectedDynamicText: {
		type: Object,
		required: false,
	},
    setSelectedDynamicText: {
        type: Function,
        required: false,
    },
    removeOuterBorder: {
        type: Boolean,
        required: false,
        default: false,
    },
    isLastRow: {
        type: Boolean,
        required: false,
        default: false,
    },
});

// Inline suppression for body outer edges when removeOuterBorder is enabled
const outerEdgeStyle = (colIdx, totalCols, isLastRow) => {
    const styles = [];
    if (colIdx === 0) styles.push("border-left-style: none !important;");
    if (colIdx === totalCols - 1) styles.push("border-right-style: none !important;");
    if (isLastRow) styles.push("border-bottom-style: none !important;");
    return styles.join(" ");
};
</script>

<style lang="scss" scoped>
/* Apply outside-edge borders on body only when the table container does NOT have no-outer-border */
.table-container:not(.no-outer-border) tr:last-child td {
    border-bottom-style: solid !important;
}
.table-container:not(.no-outer-border) tr td:first-child {
    border-left-style: solid !important;
}
.table-container:not(.no-outer-border) tr td:last-child {
    border-right-style: solid !important;
}
</style>
