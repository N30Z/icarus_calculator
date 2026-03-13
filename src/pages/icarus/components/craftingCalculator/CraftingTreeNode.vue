<template>
    <div class="node-card" :class="nodeCardClasses">
        <div class="node-icon-wrap">
            <n-image
                :width="isRoot ? 52 : 44"
                :src="`${iconAssetsUrl}/ItemIcons/${recipe?.iconPath}.png`"
                :fallback-src="`${iconAssetsUrl}/Images/question-mark.png`"
                :preview-disabled="true"
                object-fit="contain"
            />
        </div>
        <!-- Name or variant dropdown -->
        <template v-if="variants.length > 1">
            <n-select
                class="variant-select"
                size="tiny"
                :value="effectiveVariantKey"
                :options="variantOptions"
                @update:value="$emit('variantChange', $event)"
                @mousedown.stop
            />
        </template>
        <div v-else class="node-name">{{ recipe?.label ?? itemLabelMap[itemId] ?? itemId }}</div>
        <template v-if="isRoot">
            <!-- Stop mousedown so the drag system doesn't trigger on control interactions -->
            <div class="node-controls" @mousedown.stop>
                <n-input-number
                    class="node-qty-input"
                    v-model:value="item.quantity"
                    size="tiny"
                    :min="1"
                    :max="100000"
                    :step="recipe?.outputQuantity ?? 1"
                    :validator="(v) => Number.isInteger(v)"
                    @update:value="$emit('quantityChange', item)"
                />
                <n-tooltip trigger="hover">
                    <template #trigger>
                        <n-button class="node-remove" secondary type="error" size="tiny" @click="$emit('remove', item)">
                            <n-icon size="11"><Times /></n-icon>
                        </n-button>
                    </template>
                    Remove from list
                </n-tooltip>
            </div>
        </template>
        <template v-else>
            <div v-if="checkState.state === 'partial'" class="node-qty node-qty--partial">
                {{ checkState.checkedQty }}/{{ checkState.totalQty }}
            </div>
            <div v-else class="node-qty">×{{ displayQuantity }}</div>
        </template>
    </div>
</template>

<script>
import { mapState } from 'pinia';
import { Times } from '@vicons/fa';
import { useIcarusStore } from '@/store/icarus';
import { itemLabelMap } from '@/utility/icarusData';
import { GAME_ASSETS_URL, ICON_ASSETS_URL } from '@/constants/common';

export default {
    name: 'CraftingTreeNode',
    components: { Times },
    props: {
        itemId: { type: String, required: true },
        quantity: { type: Number, default: 1 },
        item: { type: Object, default: null },
        isRoot: { type: Boolean, default: false },
        checkState: { type: Object, default: () => ({ state: 'none', checkedQty: 0, totalQty: 0 }) },
        variants: { type: Array, default: () => [] },
        selectedVariantKey: { type: String, default: null },
    },
    emits: ['quantityChange', 'remove', 'variantChange'],
    data() {
        return {
            itemLabelMap,
            gameAssetsUrl: GAME_ASSETS_URL,
            iconAssetsUrl: ICON_ASSETS_URL,
        };
    },
    computed: {
        ...mapState(useIcarusStore, ['recipeData']),
        recipe() {
            return this.recipeData[this.itemId];
        },
        displayQuantity() {
            return Math.ceil(this.quantity);
        },
        nodeCardClasses() {
            return {
                'node-card--root': this.isRoot,
                'node-card--checked': this.checkState.state === 'full',
                'node-card--partial': this.checkState.state === 'partial',
            };
        },
        variantOptions() {
            return this.variants.map((v) => ({ value: v.key, label: v.label }));
        },
        effectiveVariantKey() {
            return this.selectedVariantKey ?? this.variants[0]?.key ?? null;
        },
    },
};
</script>

<style lang="scss">
.node-card {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 5px;
    padding: 8px 10px;
    border: 1px solid rgba(255, 255, 255, 0.1);
    border-radius: 10px;
    background: rgba(255, 255, 255, 0.03);
    min-width: 82px;
    max-width: 120px;
    text-align: center;
    transition: border-color 0.15s, box-shadow 0.15s;

    &:hover {
        background: rgba(255, 255, 255, 0.07);
        border-color: rgba(255, 255, 255, 0.2);
    }

    &--root {
        padding: 10px 14px;
        border-color: rgba(100, 150, 255, 0.25);
        background: rgba(100, 150, 255, 0.05);
        min-width: 110px;
        max-width: 150px;
    }

    &--checked {
        border-color: #4caf50 !important;
        box-shadow: 0 0 0 2px rgba(76, 175, 80, 0.35);
        background: rgba(76, 175, 80, 0.07) !important;
    }

    &--partial {
        border-color: #f5a623 !important;
        box-shadow: 0 0 0 2px rgba(245, 166, 35, 0.35);
        background: rgba(245, 166, 35, 0.07) !important;
    }
}

.node-icon-wrap {
    line-height: 0;
}

.node-name {
    font-size: 0.72rem;
    font-weight: 600;
    line-height: 1.25;
    word-break: break-word;
    max-width: 100%;
}

.node-qty {
    font-size: 0.85rem;
    color: #aaaacc;
    font-weight: 700;

    &--partial {
        color: #f5a623;
    }
}

.node-controls {
    display: flex;
    align-items: center;
    gap: 4px;
    margin-top: 2px;

    .node-qty-input {
        width: 5rem;
    }
}

.variant-select {
    width: 100%;
    font-size: 0.7rem;
}
</style>
