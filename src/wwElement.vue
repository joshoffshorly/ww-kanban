<template>
    <div class="ww-kanban" :style="kanbanStyle">
        <template v-if="content.uncategorizedStack">
            <wwLayoutItemContext :index="0" :item="null" :data="uncategorizedStack" is-repeat>
                <div class="ww-kanban-stack-wrapper">
                    <wwElement
                        ref="uncategorizedElement"
                        v-bind="content.stackElement"
                        :ww-props="{
                            ...stackConfig,
                            items: getVisibleItems(uncategorizedStack, getStackKey(null)),
                            stack: null,
                        }"
                        class="ww-kanban-stack"
                        :data-stack-key="getStackKey(null)"
                        :states="computedIsDragging ? ['dragging'] : []"
                    ></wwElement>
                    <button
                        v-if="hasMoreItems(getStackKey(null))"
                        class="ww-kanban-load-more ww-kanban-load-more-link"
                        @click="loadMore(getStackKey(null))"
                        :style="loadMoreButtonStyle"
                    >
                        {{ content.loadMoreText || 'Load more' }}
                    </button>
                    <div
                        v-else-if="isInfiniteScrollEnabled && getTotalItemsForKey(getStackKey(null)) > 0"
                        class="ww-kanban-end-indicator"
                        :style="endIndicatorStyle"
                    >
                        {{ content.endIndicatorText || 'All items loaded' }}
                    </div>
                </div>
            </wwLayoutItemContext>
        </template>

        <template v-for="(stack, index) in internalStacks" :key="'ww-stack-' + index">
            <wwLayoutItemContext :index="index" :item="null" is-repeat :data="stack" :repeated-items="internalStacks">
                <div class="ww-kanban-stack-wrapper">
                    <wwElement
                        ref="stackElements"
                        v-bind="content.stackElement"
                        :ww-props="{ ...stackConfig, items: getVisibleItems(stack), stack: stack.value }"
                        class="ww-kanban-stack"
                        :data-stack-key="getStackKey(stack.value)"
                        :states="computedIsDragging ? ['dragging'] : []"
                    ></wwElement>
                    <button
                        v-if="hasMoreItems(getStackKey(stack.value))"
                        class="ww-kanban-load-more ww-kanban-load-more-link"
                        @click="loadMore(getStackKey(stack.value))"
                        :style="loadMoreButtonStyle"
                    >
                        {{ content.loadMoreText || 'Load more' }}
                    </button>
                    <div
                        v-else-if="isInfiniteScrollEnabled && getTotalItemsForKey(getStackKey(stack.value)) > 0"
                        class="ww-kanban-end-indicator"
                        :style="endIndicatorStyle"
                    >
                        {{ content.endIndicatorText || 'All items loaded' }}
                    </div>
                </div>
            </wwLayoutItemContext>
        </template>
    </div>
</template>

<script>
import { provide, reactive, ref, watch, computed } from "vue";

const UNCATEGORIZED_KEY = "__uncategorized__";

export default {
    props: {
        content: { type: Object, required: true },
        uid: { type: String, required: true },
        /* wwEditor:start */
        wwElementState: { type: Object, required: true },
        wwEditorState: { type: Object, required: true },
        /* wwEditor:end */
    },
    emits: ["trigger-event", "update:content:effect"],
    setup(props, { emit }) {
        const internalStacks = ref([]);
        const uncategorizedStack = reactive({
            label: "Uncategorized",
            value: null,
            items: [],
        });

        // Make visibleCounts reactive in setup for better reactivity
        const visibleCounts = reactive({});

        provide("customHandler", (change, { stack: stackValue, updatedStackItems }) => {
            if (change.moved) {
                emit("trigger-event", {
                    name: "item:moved",
                    event: {
                        item: change.moved.element,
                        from: stackValue,
                        to: stackValue,
                        oldIndex: change.moved.oldIndex,
                        newIndex: change.moved.newIndex,
                        updatedList: updatedStackItems,
                    },
                });
            }

            if (change.added) {
                emit("trigger-event", {
                    name: "item:moved",
                    event: {
                        item: change.added.element,
                        from: wwLib.resolveObjectPropertyPath(change.added.element, props.content.stackedBy),
                        to: stackValue,
                        oldIndex: null,
                        newIndex: change.added.newIndex,
                        updatedList: updatedStackItems,
                    },
                });
            }
        });

        const isDraggingManager = reactive({});
        provide("customDragHandler", (isDragging, { stack }) => (isDraggingManager[stack] = isDragging));

        const { setValue: setDrag } = wwLib.wwVariable.useComponentVariable({
            uid: props.uid,
            name: "isDragging",
            type: "boolean",
            defaultValue: false,
            readonly: true,
        });
        watch(
            isDraggingManager,
            (value) => {
                setDrag(Object.values(value).some((isDragging) => isDragging));
            },
            { deep: true }
        );

        const isDragging = computed(() => Object.values(isDraggingManager).some((isDragging) => isDragging));

        const css = computed(() => `* { cursor: ${props.content.draggingCursor || "grabbing"} !important; }`);
        const styletag = wwLib.getFrontDocument().createElement("style");

        watch(
            isDragging,
            (value) => {
                if (value) {
                    styletag.appendChild(wwLib.getFrontDocument().createTextNode(css.value));
                    wwLib.getFrontDocument().body.appendChild(styletag);
                } else {
                    styletag.remove();
                }
            },
            { deep: true }
        );

        return { internalStacks, uncategorizedStack, isDragging, visibleCounts };
    },
    computed: {
        // Add a computed property that's accessible in the component
        computedIsDragging() {
            return this.isDragging;
        },
        stacks() {
            return wwLib.wwUtils.getDataFromCollection(this.content.stacks) || [];
        },
        items() {
            return wwLib.wwUtils.getDataFromCollection(this.content.items) || [];
        },
        stackConfig() {
            return {
                sortable: this.content.sortable,
                group: "kanban-" + this.uid,
                itemKey: this.content.itemKey,
                handle: this.content.customDragHandle ? this.content.handleClass || "draggable" : null,
                readonly: this.content.readonly,
            };
        },
        kanbanStyle() {
            return {
                "--wrap-stacks": this.content.wrapStacks ? "wrap" : "nowrap",
            };
        },
        loadMoreButtonStyle() {
            return {
                color: this.content.loadMoreTextColor || '#007bff',
                padding: this.content.loadMorePadding || '8px 0',
                backgroundColor: 'transparent',
                border: 'none',
                cursor: 'pointer',
                fontSize: this.content.loadMoreFontSize || '14px',
                fontWeight: this.content.loadMoreFontWeight || 'normal',
                textDecoration: 'none',
                width: '100%',
                textAlign: 'center',
                marginTop: this.content.loadMoreMarginTop || '8px',
            };
        },
        endIndicatorStyle() {
            return {
                color: this.content.endIndicatorColor || '#6c757d',
                padding: '8px 0',
                fontSize: this.content.endIndicatorFontSize || '12px',
                textAlign: 'center',
                fontStyle: 'italic',
                opacity: '0.7',
                width: '100%',
                marginTop: this.content.endIndicatorMarginTop || '8px',
            };
        },
        isReadonly() {
            /* wwEditor:start */
            if (this.wwEditorState.isSelected) {
                return this.wwElementState.states.includes("readonly");
            }
            /* wwEditor:end */
            return this.content.readonly;
        },
        isInfiniteScrollEnabled() {
            return !!this.content.enableInfiniteScroll;
        },
        pageSize() {
            const size = Number(this.content.pageSize);
            return Number.isFinite(size) && size > 0 ? size : 20;
        },
    },
    watch: {
        "content.stackValue"() {
            this.refreshStacks();
        },
        "content.stackedBy"() {
            this.refreshStacks();
        },
        "content.sortedBy"() {
            this.refreshStacks();
        },
        "content.sortOrder"() {
            this.refreshStacks();
        },
        "content.stacks": {
            handler() {
                this.refreshStacks();
            },
            deep: true,
        },
        stacks() {
            this.refreshStacks();
        },
        items: {
            handler() {
                this.refreshStacks();
            },
            deep: true,
        },
        isReadonly: {
            immediate: true,
            handler(value) {
                if (value) {
                    this.$emit("add-state", "readonly");
                } else {
                    this.$emit("remove-state", "readonly");
                }
            },
        },
        "content.enableInfiniteScroll"(value) {
            if (value) {
                this.resetVisibleCounts();
                this.syncVisibleCounts();
            }
        },
        "content.pageSize"() {
            if (this.isInfiniteScrollEnabled) {
                this.resetVisibleCounts();
                this.syncVisibleCounts();
            }
        },
    },
    methods: {
        getStackKey(stackValue) {
            return stackValue == null ? UNCATEGORIZED_KEY : String(stackValue);
        },
        getVisibleItems(stack, stackKey) {
            if (!this.isInfiniteScrollEnabled) return stack.items;
            const key = stackKey || this.getStackKey(stack.value);
            const visibleCount = this.visibleCounts[key] ?? this.pageSize;
            return stack.items.slice(0, visibleCount);
        },
        hasMoreItems(stackKey) {
            if (!this.isInfiniteScrollEnabled) return false;
            const totalItems = this.getTotalItemsForKey(stackKey);
            const visibleCount = this.visibleCounts[stackKey] ?? this.pageSize;
            return visibleCount < totalItems;
        },
        loadMore(stackKey) {
            if (!this.isInfiniteScrollEnabled || !stackKey) return;
            
            const totalItems = this.getTotalItemsForKey(stackKey);
            const current = this.visibleCounts[stackKey] ?? this.pageSize;
            
            if (current < totalItems) {
                const newCount = Math.min(totalItems, current + this.pageSize);
                this.visibleCounts[stackKey] = newCount;
                
                // Emit event for load more action
                this.$emit("trigger-event", {
                    name: "load:more",
                    event: {
                        stackKey,
                        currentCount: current,
                        newCount: newCount,
                        totalItems: totalItems,
                    },
                });
            }
        },
        resetVisibleCounts() {
            Object.keys(this.visibleCounts).forEach((key) => delete this.visibleCounts[key]);
        },
        syncVisibleCounts() {
            if (!this.isInfiniteScrollEnabled) return;
            const pageSize = this.pageSize;

            this.internalStacks.forEach((stack) => {
                const key = this.getStackKey(stack.value);
                if (!this.visibleCounts[key]) {
                    this.visibleCounts[key] = Math.min(stack.items.length, pageSize);
                }
            });

            if (this.content.uncategorizedStack) {
                const key = this.getStackKey(null);
                if (!this.visibleCounts[key]) {
                    this.visibleCounts[key] = Math.min(this.uncategorizedStack.items.length, pageSize);
                }
            }
        },
        getTotalItemsForKey(stackKey) {
            if (stackKey === this.getStackKey(null)) return this.uncategorizedStack.items.length;
            const stack = this.internalStacks.find((item) => this.getStackKey(item.value) === stackKey);
            return stack ? stack.items.length : 0;
        },
        refreshStacks() {
            this.internalStacks = this.stacks
                .map((stack) => ({
                    label: wwLib.resolveObjectPropertyPath(stack, this.content.stackLabel || "label") ?? "",
                    value: wwLib.resolveObjectPropertyPath(stack, this.content.stackValue || "value") ?? "",
                }))
                .map((stack) => ({
                    ...stack,
                    items: this.items
                        .filter((item) => wwLib.resolveObjectPropertyPath(item, this.content.stackedBy) === stack.value)
                        .sort((a, b) => {
                            if (!this.content.sortedBy) return 0;
                            const valueA = wwLib.resolveObjectPropertyPath(a, this.content.sortedBy);
                            const valueB = wwLib.resolveObjectPropertyPath(b, this.content.sortedBy);
                            if (this.content.sortOrder === "asc") {
                                return valueA > valueB ? 1 : -1;
                            } else {
                                return valueA > valueB ? -1 : 1;
                            }
                        }),
                }));
            const stacksList = this.stacks.map((stack) =>
                wwLib.resolveObjectPropertyPath(stack, this.content.stackValue || "value")
            );
            this.uncategorizedStack.items = this.items.filter(
                (item) => !stacksList.includes(wwLib.resolveObjectPropertyPath(item, this.content.stackedBy))
            );
            this.syncVisibleCounts();
        },
        /* wwEditor:start */
        getTestEvent() {
            if (!this.internalStacks.length) throw new Error("No stack found");
            if (!this.items.length) throw new Error("No item found");
            return {
                item: this.items[0],
                from: this.internalStacks[0].value,
                to: this.internalStacks[0].value,
                oldIndex: 0,
                newIndex: 1,
                updatedList: this.items,
            };
        },
        /* wwEditor:end */
    },
    created() {
        this.refreshStacks();
    },
};
</script>

<style lang="scss" scoped>
.ww-kanban {
    flex-direction: row;
    flex-wrap: var(--wrap-stacks);
}

.ww-kanban-stack-wrapper {
    display: flex;
    flex-direction: column;
    height: 100%;
}

.ww-kanban-load-more-link {
    &:hover {
        text-decoration: underline;
        opacity: 0.8;
    }
    
    &:active {
        opacity: 0.6;
    }
}

.ww-kanban-end-indicator {
    user-select: none;
}
</style>
