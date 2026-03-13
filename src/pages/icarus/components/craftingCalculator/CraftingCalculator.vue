<template>
    <div>
        <div v-if="!isLoadingRecipes && tab.items.length > 0">
            <div class="flex align-items-center">
                <h3>Items</h3>
                <n-tooltip trigger="hover">
                    <template #trigger>
                        <n-button class="ml-3" size="tiny" secondary circle type="default" @click="sortInputs">
                            <n-icon size="13"><SortAlphaDown /></n-icon>
                        </n-button>
                    </template>
                    Sort items alphabetically
                </n-tooltip>
            </div>

            <div class="p-1 mb-3">
                <div class="flex align-items-center mb-1">
                    <n-switch class="mr-2" v-model:value="settings.includeSubComponents" size="small" />
                    <span>Include sub-components</span>
                </div>
                <div class="flex align-items-center ml-3">
                    <n-switch class="mr-2" v-model:value="settings.includeStationComponents" size="small" />
                    <span>Include station components</span>
                </div>
            </div>

            <!-- Interactive canvas -->
            <div
                class="canvas-viewport"
                ref="viewport"
                :style="{ cursor: isPanning ? 'grabbing' : draggingNode ? 'grabbing' : 'default' }"
                @wheel.prevent="onWheel"
                @mousedown="onViewportMouseDown"
                @mousemove="onMouseMove"
                @mouseup="onMouseUp"
                @mouseleave="onMouseUp"
            >
                <div class="canvas-content" :style="canvasTransformStyle">
                    <!-- Connector lines -->
                    <svg
                        :width="canvasDimensions.width"
                        :height="canvasDimensions.height"
                        style="position: absolute; top: 0; left: 0; pointer-events: none; overflow: visible;"
                    >
                        <line
                            v-for="line in allConnectorLines"
                            :key="line.key"
                            :x1="line.x1" :y1="line.y1"
                            :x2="line.x2" :y2="line.y2"
                            stroke="rgba(255,255,255,0.3)"
                            stroke-width="1.5"
                            stroke-linecap="round"
                        />
                    </svg>

                    <!-- Level 1: root items -->
                    <div
                        v-for="item in tab.items"
                        :key="item.id"
                        class="node-wrapper"
                        :style="nodeStyle('root', item.id)"
                        @mousedown="startDrag($event, 'root', item.id)"
                    >
                        <crafting-tree-node
                            :item-id="item.id"
                            :item="item"
                            :is-root="true"
                            :check-state="nodeCheckStates['root:' + item.id]"
                            :variants="getVariants(item.id)"
                            :selected-variant-key="getSelectedVariantKey(item.id)"
                            @quantityChange="triggerCalc"
                            @remove="onRemoveItem"
                            @variantChange="(key) => setVariant(item.id, key)"
                        />
                    </div>

                    <!-- Component levels (L2–L5) -->
                    <template v-for="(levelNodes, levelIdx) in treeLevels" :key="'level-' + levelIdx">
                        <div
                            v-for="node in levelNodes"
                            :key="node.id"
                            class="node-wrapper"
                            :style="nodeStyle(levelKeyOf(levelIdx), node.id)"
                            @mousedown="startDrag($event, levelKeyOf(levelIdx), node.id)"
                        >
                            <crafting-tree-node
                                :item-id="node.id"
                                :quantity="node.quantity"
                                :check-state="nodeCheckStates[levelKeyOf(levelIdx) + ':' + node.id]"
                                :variants="getVariants(node.id)"
                                :selected-variant-key="getSelectedVariantKey(node.id)"
                                @variantChange="(key) => setVariant(node.id, key)"
                            />
                        </div>
                    </template>
                </div>
            </div>

            <div class="flex align-items-center mt-4">
                <h3>Crafting Stations</h3>
            </div>
            <div>
                <div v-for="componentName in requiredCraftingStations" class="recipe-item stations pl-0 flex align-items-center">
                    <div class="flex align-items-center">
                        <n-image
                            class="icon"
                            width="32"
                            :src="`${gameAssetsUrl}/ItemIcons/${recipeData[componentName]?.iconPath}.png`"
                            :fallback-src="`${gameAssetsUrl}/Images/question-mark.png`"
                            :preview-disabled="true"
                        />
                        <div class="label">{{ recipeData[componentName]?.label ?? itemLabelMap[componentName] ?? componentName }}</div>
                    </div>
                </div>
            </div>
        </div>

        <div v-else class="mb-3">
            <h3>No Items</h3>
            <n-text type="info">You haven't added any items to this list yet.</n-text>
        </div>
    </div>
</template>

<script>
import debounce from 'debounce';
import { mapActions, mapGetters, mapState } from 'pinia';
import { SortAlphaDown } from '@vicons/fa';

import CraftingTreeNode from './CraftingTreeNode.vue';
import { useIcarusStore } from '@/store/icarus';
import { itemLabelMap } from '@/utility/icarusData';
import { GAME_ASSETS_URL } from '@/constants/common';

// Layout constants (px) — used for initial placement and line endpoints
const CARD_W = 130;
const CARD_H_ROOT = 142; // icon(52) + gap(5) + name(~18) + gap(5) + controls(~32) + padding(20) + border
const CARD_H = 112;      // icon(44) + gap(5) + name(~18) + gap(5) + qty(~20) + padding(16) + border
const GAP_H = 16;
const GAP_V = 80;
const CANVAS_PAD = 40;

export default {
    name: 'CraftingToolCalculator',
    components: { CraftingTreeNode, SortAlphaDown },
    props: {
        tab: { type: Object, required: true },
    },
    data() {
        return {
            itemLabelMap,
            gameAssetsUrl: GAME_ASSETS_URL,
            requiredCraftingStations: [],
            nodePositions: {},
            checkedNodes: {},
            selectedVariants: {},
            zoom: 1,
            panX: 0,
            panY: 0,
            isPanning: false,
            panStart: { x: 0, y: 0 },
            draggingNode: null,
        };
    },
    computed: {
        ...mapState(useIcarusStore, ['recipeData', 'recipeByOutput', 'itemTableData', 'isLoadingRecipes', 'settings']),
        ...mapGetters(useIcarusStore, ['includeSubComponents', 'includeStationComponents']),

        // Returns a function (itemId) => recipe, respecting selectedVariants
        effectiveRecipeGetter() {
            const selectedVariants = this.selectedVariants;
            const recipeData = this.recipeData;
            const recipeByOutput = this.recipeByOutput;
            return (itemId) => {
                // Direct recipe keyed by itemId (most common case)
                const direct = recipeData[itemId];
                if (direct) {
                    const outputId = direct.outputItemId ?? itemId;
                    const selectedKey = selectedVariants[outputId];
                    if (selectedKey && recipeData[selectedKey]) return recipeData[selectedKey];
                    return direct;
                }
                // Lookup by output item ID (e.g. "Composite_Paste" when recipe name is "Composite_Paste_Plat")
                const variants = recipeByOutput?.[itemId];
                if (variants?.length) {
                    const selectedKey = selectedVariants[itemId];
                    return variants.find((v) => v.id === selectedKey) ?? variants[0];
                }
                return null;
            };
        },

        // Builds up to 5 levels of merged components (L2–L5).
        // Items that appear in multiple levels are moved to their deepest level,
        // so connector lines can skip levels without duplicating nodes.
        treeLevels() {
            const MAX_DEPTH = 4; // 4 sub-levels → 5 rows total (L1 root + L2–L5)
            const getRecipe = this.effectiveRecipeGetter;

            const buildRawLevel = (parentItems) => {
                const map = new Map();
                for (const parent of parentItems) {
                    const recipe = getRecipe(parent.id);
                    if (!recipe?.inputs?.length) continue;
                    const multiplier = parent.quantity / (recipe.outputQuantity ?? 1);
                    for (const input of recipe.inputs) {
                        if (map.has(input.id)) {
                            const e = map.get(input.id);
                            e.quantity += input.quantity * multiplier;
                            if (!e.parentIds.includes(parent.id)) e.parentIds.push(parent.id);
                        } else {
                            map.set(input.id, { id: input.id, quantity: input.quantity * multiplier, parentIds: [parent.id] });
                        }
                    }
                }
                return map;
            };

            // Build naive levels allowing duplicates across levels
            const naiveLevels = [];
            const rootAsItems = (this.tab.items || []).map((i) => ({ id: i.id, quantity: i.quantity ?? 1 }));
            const l2 = buildRawLevel(rootAsItems);
            if (!l2.size) return [];
            naiveLevels.push(l2);

            if (this.includeSubComponents) {
                for (let d = 1; d < MAX_DEPTH; d++) {
                    const next = buildRawLevel(Array.from(naiveLevels[d - 1].values()));
                    if (!next.size) break;
                    naiveLevels.push(next);
                }
            }

            // Merge duplicates: each item is placed at its deepest level,
            // with quantities and parentIds accumulated across all levels.
            const itemDeepestLevel = new Map(); // id -> deepest levelIdx
            const itemMerged = new Map(); // id -> merged {id, quantity, parentIds}

            for (let levelIdx = 0; levelIdx < naiveLevels.length; levelIdx++) {
                for (const [id, node] of naiveLevels[levelIdx]) {
                    itemDeepestLevel.set(id, levelIdx); // iterating in order keeps the deepest
                    if (itemMerged.has(id)) {
                        const m = itemMerged.get(id);
                        m.quantity += node.quantity;
                        for (const pid of node.parentIds) {
                            if (!m.parentIds.includes(pid)) m.parentIds.push(pid);
                        }
                    } else {
                        itemMerged.set(id, { id, quantity: node.quantity, parentIds: [...node.parentIds] });
                    }
                }
            }

            // Distribute merged items into final level slots
            const finalLevels = Array.from({ length: naiveLevels.length }, () => []);
            for (const [id, levelIdx] of itemDeepestLevel) {
                const m = itemMerged.get(id);
                finalLevels[levelIdx].push({ ...m, quantity: Math.ceil(m.quantity) });
            }

            // Trim trailing empty levels
            while (finalLevels.length && !finalLevels[finalLevels.length - 1].length) finalLevels.pop();

            return finalLevels;
        },

        mergedDirectComponents() {
            return this.treeLevels[0] ?? [];
        },

        // Derives check state for every node: { state: 'none'|'partial'|'full', checkedQty, totalQty }
        nodeCheckStates() {
            const states = {};
            const checkedNodes = this.checkedNodes;
            const treeLevels = this.treeLevels;
            const getRecipe = this.effectiveRecipeGetter;

            // Build lookups: itemId -> levelKey and itemId -> quantity
            const levelKeyByItemId = {};
            const quantityByItemId = {};
            for (const item of this.tab.items || []) {
                levelKeyByItemId[item.id] = 'root';
                quantityByItemId[item.id] = item.quantity ?? 1;
            }
            for (let i = 0; i < treeLevels.length; i++) {
                const lk = `l${i + 2}`;
                for (const node of treeLevels[i]) {
                    levelKeyByItemId[node.id] = lk;
                    quantityByItemId[node.id] = node.quantity;
                }
            }

            // Root nodes: direct toggle
            for (const item of this.tab.items || []) {
                const key = `root:${item.id}`;
                states[key] = { state: !!checkedNodes[key] ? 'full' : 'none', checkedQty: 0, totalQty: 0 };
            }

            // Each component level (parents are always at shallower levels, so order is safe)
            for (let levelIdx = 0; levelIdx < treeLevels.length; levelIdx++) {
                const lk = `l${levelIdx + 2}`;

                for (const node of treeLevels[levelIdx]) {
                    const key = `${lk}:${node.id}`;
                    if (!!checkedNodes[key]) {
                        states[key] = { state: 'full', checkedQty: 0, totalQty: 0 };
                        continue;
                    }

                    const checkedParentCount = node.parentIds.filter((pid) => {
                        const parentLk = levelKeyByItemId[pid];
                        return parentLk && states[`${parentLk}:${pid}`]?.state === 'full';
                    }).length;
                    const totalParents = node.parentIds.length;

                    if (checkedParentCount === 0) {
                        states[key] = { state: 'none', checkedQty: 0, totalQty: 0 };
                    } else if (checkedParentCount === totalParents) {
                        states[key] = { state: 'full', checkedQty: 0, totalQty: 0 };
                    } else {
                        let checkedQty = 0;
                        const totalQty = node.quantity;
                        for (const pid of node.parentIds) {
                            const parentLk = levelKeyByItemId[pid];
                            if (!parentLk) continue;
                            const parentQty = quantityByItemId[pid] ?? 1;
                            const recipe = getRecipe(pid);
                            if (!recipe) continue;
                            const multiplier = parentQty / (recipe.outputQuantity ?? 1);
                            const inputEntry = recipe.inputs.find((inp) => inp.id === node.id);
                            if (!inputEntry) continue;
                            if (states[`${parentLk}:${pid}`]?.state === 'full') {
                                checkedQty += Math.ceil(inputEntry.quantity * multiplier);
                            }
                        }
                        states[key] = { state: 'partial', checkedQty, totalQty };
                    }
                }
            }

            return states;
        },

        canvasDimensions() {
            let maxX = CANVAS_PAD;
            let maxY = CANVAS_PAD;
            for (const [key, pos] of Object.entries(this.nodePositions)) {
                const h = key.startsWith('root:') ? CARD_H_ROOT : CARD_H;
                maxX = Math.max(maxX, pos.x + CARD_W + CANVAS_PAD);
                maxY = Math.max(maxY, pos.y + h + CANVAS_PAD);
            }
            return { width: Math.max(600, maxX), height: Math.max(400, maxY) };
        },

        canvasTransformStyle() {
            return {
                transform: `translate(${this.panX}px, ${this.panY}px) scale(${this.zoom})`,
                transformOrigin: '0 0',
                position: 'relative',
                width: `${this.canvasDimensions.width}px`,
                height: `${this.canvasDimensions.height}px`,
            };
        },

        allConnectorLines() {
            const lines = [];

            // Build lookup: itemId -> levelKey (parents can be at any level)
            const levelKeyByItemId = {};
            for (const item of this.tab.items || []) levelKeyByItemId[item.id] = 'root';
            for (let i = 0; i < this.treeLevels.length; i++) {
                const lk = `l${i + 2}`;
                for (const node of this.treeLevels[i]) levelKeyByItemId[node.id] = lk;
            }

            for (let levelIdx = 0; levelIdx < this.treeLevels.length; levelIdx++) {
                const lk = `l${levelIdx + 2}`;
                for (const node of this.treeLevels[levelIdx]) {
                    const childPos = this.nodePositions[`${lk}:${node.id}`];
                    if (!childPos) continue;
                    for (const parentId of node.parentIds) {
                        const parentLk = levelKeyByItemId[parentId];
                        if (!parentLk) continue;
                        const parentPos = this.nodePositions[`${parentLk}:${parentId}`];
                        if (!parentPos) continue;
                        const parentH = parentLk === 'root' ? CARD_H_ROOT : CARD_H;
                        lines.push({
                            key: `${parentLk}:${parentId}->${lk}:${node.id}`,
                            x1: parentPos.x + CARD_W / 2,
                            y1: parentPos.y + parentH,
                            x2: childPos.x + CARD_W / 2,
                            y2: childPos.y,
                        });
                    }
                }
            }
            return lines;
        },
    },

    watch: {
        'tab.items': {
            handler() {
                this.syncNodePositions();
                this.triggerCalc();
            },
            deep: true,
        },
        includeSubComponents(newVal) {
            if (!newVal) this.setIncludeStationComponents(false);
            this.triggerCalc();
            this.syncNodePositions();
        },
        includeStationComponents(newVal) {
            if (newVal) this.setIncludeSubComponents(true);
            this.triggerCalc();
        },
    },

    mounted() {
        this.syncNodePositions();
    },

    methods: {
        ...mapActions(useIcarusStore, ['setIncludeSubComponents', 'setIncludeStationComponents']),

        levelKeyOf(levelIdx) {
            return `l${levelIdx + 2}`;
        },

        getVariants(itemId) {
            // Resolve to output item ID in case itemId is a recipe name (e.g. "Composite_Paste_Plat" → "Composite_Paste")
            const outputId = this.recipeData[itemId]?.outputItemId ?? itemId;
            const variants = this.recipeByOutput?.[outputId];
            if (variants && variants.length > 1) {
                return variants.map((v) => ({ key: v.id, label: v.label }));
            }
            return [];
        },

        getSelectedVariantKey(itemId) {
            const outputId = this.recipeData[itemId]?.outputItemId ?? itemId;
            return this.selectedVariants[outputId] ?? null;
        },

        setVariant(itemId, recipeKey) {
            const outputId = this.recipeData[itemId]?.outputItemId ?? itemId;
            this.selectedVariants[outputId] = recipeKey;
            this.syncNodePositions();
            this.triggerCalc();
        },

        sortInputs() {
            this.tab.items.sort((a, b) =>
                this.recipeData[a.id].label.localeCompare(this.recipeData[b.id].label)
            );
        },

        onRemoveItem(item) {
            const idx = (this.tab.items || []).findIndex((i) => i.id === item.id);
            if (idx > -1) this.tab.items.splice(idx, 1);
            this.triggerCalc();
        },

        syncNodePositions() {
            const newPositions = { ...this.nodePositions };
            const rootItems = this.tab.items || [];
            const treeLevels = this.treeLevels;

            // Build valid key set
            const validKeys = new Set(rootItems.map((i) => `root:${i.id}`));
            treeLevels.forEach((level, idx) => {
                const lk = `l${idx + 2}`;
                level.forEach((n) => validKeys.add(`${lk}:${n.id}`));
            });

            // Remove stale keys
            for (const key of Object.keys(newPositions)) {
                if (!validKeys.has(key)) delete newPositions[key];
            }

            // Compute initial positions for new nodes only
            const allRows = [rootItems, ...treeLevels];
            const maxLen = Math.max(...allRows.map((r) => r.length), 1);
            const totalW = maxLen * CARD_W + (maxLen - 1) * GAP_H + 2 * CANVAS_PAD;

            allRows.forEach((row, rowIdx) => {
                const lk = rowIdx === 0 ? 'root' : `l${rowIdx + 1}`;
                const yOffset = rowIdx === 0
                    ? CANVAS_PAD
                    : CANVAS_PAD + CARD_H_ROOT + GAP_V + (rowIdx - 1) * (CARD_H + GAP_V);
                const rowW = row.length * CARD_W + Math.max(0, row.length - 1) * GAP_H;
                const startX = (totalW - rowW) / 2;
                row.forEach((item, i) => {
                    const key = `${lk}:${item.id}`;
                    if (!newPositions[key]) {
                        newPositions[key] = { x: startX + i * (CARD_W + GAP_H), y: yOffset };
                    }
                });
            });

            this.nodePositions = newPositions;
        },

        nodeStyle(level, id) {
            const pos = this.nodePositions[`${level}:${id}`];
            if (!pos) return { display: 'none' };
            const isActive = this.draggingNode?.key === `${level}:${id}`;
            return {
                position: 'absolute',
                left: `${pos.x}px`,
                top: `${pos.y}px`,
                cursor: isActive ? 'grabbing' : 'grab',
                userSelect: 'none',
                zIndex: isActive ? 100 : 1,
            };
        },

        startDrag(e, level, id) {
            e.stopPropagation();
            const key = `${level}:${id}`;
            const pos = this.nodePositions[key];
            if (!pos) return;
            this.draggingNode = {
                key,
                startMouseX: e.clientX,
                startMouseY: e.clientY,
                startNodeX: pos.x,
                startNodeY: pos.y,
                hasMoved: false,
            };
        },

        onViewportMouseDown(e) {
            this.isPanning = true;
            this.panStart = { x: e.clientX - this.panX, y: e.clientY - this.panY };
        },

        onMouseMove(e) {
            if (this.draggingNode) {
                const dx = (e.clientX - this.draggingNode.startMouseX) / this.zoom;
                const dy = (e.clientY - this.draggingNode.startMouseY) / this.zoom;
                if (Math.abs(dx) > 3 || Math.abs(dy) > 3) this.draggingNode.hasMoved = true;
                const { key, startNodeX, startNodeY } = this.draggingNode;
                this.nodePositions[key] = { x: startNodeX + dx, y: startNodeY + dy };
            } else if (this.isPanning) {
                this.panX = e.clientX - this.panStart.x;
                this.panY = e.clientY - this.panStart.y;
            }
        },

        onMouseUp() {
            if (this.draggingNode) {
                if (!this.draggingNode.hasMoved) {
                    const key = this.draggingNode.key;
                    this.checkedNodes[key] = !this.checkedNodes[key];
                }
                this.draggingNode = null;
            }
            this.isPanning = false;
        },

        onWheel(e) {
            const factor = e.deltaY > 0 ? 0.9 : 1.1;
            const newZoom = Math.min(3, Math.max(0.15, this.zoom * factor));
            const rect = this.$refs.viewport.getBoundingClientRect();
            const mx = e.clientX - rect.left;
            const my = e.clientY - rect.top;
            const ratio = newZoom / this.zoom;
            this.panX = mx - (mx - this.panX) * ratio;
            this.panY = my - (my - this.panY) * ratio;
            this.zoom = newZoom;
        },

        triggerCalc: debounce(function () {
            this.calculateRequiredStations();
        }, 100),

        calculateRequiredStations() {
            const selectedItems = this.tab.items || [];
            const getRecipe = this.effectiveRecipeGetter;
            const recipeData = this.recipeData;
            const includeSubComponents = this.includeSubComponents;
            const includeStationComponents = this.includeStationComponents;
            const requiredItemData = {};
            const requiredCraftingStations = new Set();

            function sumInputsForItems(items = [], dataMap = {}) {
                items.forEach((item) => {
                    const itemData = getRecipe(item.id);
                    if (!itemData) return;
                    const multiplier = (item.quantity ?? 1) / itemData.outputQuantity;
                    (itemData.inputs || []).forEach((input) => {
                        dataMap[input.id] = (dataMap[input.id] ?? 0) + input.quantity * multiplier;
                        if (includeSubComponents && getRecipe(input.id)) {
                            sumInputsForItems([{ id: input.id, quantity: input.quantity * multiplier }], dataMap);
                        }
                    });
                });
                return dataMap;
            }

            function populateCraftingStations() {
                [...selectedItems.map((i) => i.id), ...Object.keys(requiredItemData)].forEach((name) => {
                    const d = getRecipe(name);
                    if (d?.preferredSource) requiredCraftingStations.add(d.preferredSource);
                });
            }

            sumInputsForItems(selectedItems, requiredItemData);
            populateCraftingStations();

            if (includeStationComponents) {
                sumInputsForItems([...requiredCraftingStations].map((s) => ({ id: s })), requiredItemData);
                populateCraftingStations();
            }

            this.requiredCraftingStations = [...requiredCraftingStations].sort((a, b) => {
                const al = recipeData[a]?.label ?? itemLabelMap[a] ?? a;
                const bl = recipeData[b]?.label ?? itemLabelMap[b] ?? b;
                return al.localeCompare(bl);
            });
        },
    },
};
</script>

<style lang="scss">
.canvas-viewport {
    height: 500px;
    overflow: hidden;
    border: 1px solid rgba(255, 255, 255, 0.1);
    border-radius: 8px;
    background: rgba(0, 0, 0, 0.15);
    margin-bottom: 1rem;
    position: relative;
}

.canvas-content {
    will-change: transform;
}

.node-wrapper {
    // position and cursor set dynamically via nodeStyle()
}

.recipe-item {
    min-height: 40px;
    padding: 0.4rem 0.3rem;

    &.stations {
        min-height: 36px;
    }

    .icon {
        margin: 0 0.5rem 0 0;
    }

    .label {
        font-weight: 600;
        line-height: 18px;
    }
}
</style>
