<template>
    <div class="input-group">
        <label>{{ title }}:</label>
        <div class="input-group margin-bottom">
            <select class="select select--content-spacing" @change="emitSelected">
                <option selected disabled>
                    Select a category
                </option>
                <option v-for="(key, index) in selectableCategories" :key="`category--${key}-${index}`">
                    {{ key }}
                </option>
            </select>
        </div>
        <div class="tags has-addons" v-if="selectedCategories.length > 0">
            <span class="margin-right" v-for="(key, index) in selectedCategories" :key="`${key}-${index}`">
                <a href="#" @click="emitDeselected(key)">
                    <div class="tag has-addons">
                        <span>{{ key }}</span>
                    </div>
                    <span class="tag is-delete is-danger">&nbsp;</span>
                </a>
            </span>
        </div>
        <div class="field has-addons" v-else>
            <span class="tags">
                <span class="tag">No categories selected</span>
            </span>
        </div>
    </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from 'vue-property-decorator';

@Component
export default class ChangeSelectorModal extends Vue {

    @Prop({required: true})
    readonly title!: string;

    @Prop({required: true})
    readonly selectedCategories!: string[]

    @Prop({required: true})
    readonly selectableCategories!: string[]

    emitSelected(event: Event) {
        this.$emit("selected-category", event);
    }

    emitDeselected(key: string) {
        this.$emit("deselected-category", key);
    }

}
</script>
