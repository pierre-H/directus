<template>
	<v-list :mandatory="false" @toggle="loadFieldRelations($event.value)">
		<slot name="prepend" />
		<v-list-item v-if="fieldsCount > 20">
			<v-list-item-content>
				<v-input v-model="search" autofocus small :placeholder="t('search')">
					<template #append>
						<v-icon small name="search" />
					</template>
				</v-input>
			</v-list-item-content>
		</v-list-item>

		<v-field-list-item
			v-for="fieldNode in treeList"
			:key="fieldNode.field"
			:field="fieldNode"
			:search="search"
			:include-functions="includeFunctions"
			@add="$emit('select-field', $event)"
		/>
	</v-list>
</template>

<script lang="ts" setup>
import { useFieldTree } from '@/composables/use-field-tree';
import { Field } from '@directus/shared/types';
import { computed, ref, toRefs, watch } from 'vue';
import { useI18n } from 'vue-i18n';
import VFieldListItem from './v-field-list-item.vue';
import { debounce } from 'lodash';
import { useFieldsStore } from '@/stores';

interface Props {
	collection: string;
	field?: string;
	disabledFields?: string[];
	includeFunctions?: boolean;
}

const props = withDefaults(defineProps<Props>(), {
	disabledFields: () => [],
	includeFunctions: false,
	field: undefined,
});

defineEmits(['select-field']);

const fieldsStore = useFieldsStore();

const { collection } = toRefs(props);

const fieldsCount = computed(() => fieldsStore.getFieldsForCollection(collection.value)?.length ?? 0);

const search = ref('');

const { treeList: treeListOriginal, loadFieldRelations, refresh } = useFieldTree(collection, undefined, filter);

const debouncedRefresh = debounce(() => refresh(), 250);

watch(search, () => debouncedRefresh());

const { t } = useI18n();

const treeList = computed(() => {
	const list = treeListOriginal.value.map(setDisabled);

	if (props.field) return list.filter((fieldNode) => fieldNode.field === props.field);

	return list;

	function setDisabled(
		field: typeof treeListOriginal.value[number]
	): typeof treeListOriginal.value[number] & { disabled: boolean } {
		let disabled = false;

		if (props.disabledFields?.includes(field.key)) disabled = true;

		return {
			...field,
			disabled,
			children: field.children?.map(setDisabled),
		};
	}
});

function filter(field: Field): boolean {
	if (!search.value) return true;
	const children = fieldsStore.getFieldGroupChildren(collection.value, field.field);
	return children?.some((field) => matchesSearch(field)) || matchesSearch(field);

	function matchesSearch(field: Field) {
		return (
			field.field.toLowerCase().includes(search.value.toLowerCase()) ||
			field.name.toLowerCase().includes(search.value.toLowerCase())
		);
	}
}
</script>

<style lang="scss" scoped>
.v-list {
	--v-list-min-width: 300px;
}
</style>
