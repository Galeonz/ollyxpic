<template>
    <div class="equipments row">
        <!-- Equipment slots -->
        <div class="col-md-2" id="slots">
            <panel class="panel-slots">
                <div class="slots">
                    <div class="slot" :class="[index, slot.id ? 'active' : '']" v-for="slot, index in slots">
                        <slot-item :item.sync="slot" :index="index" @remove="removeSlot"/>
                    </div>
                </div>
            </panel>
        </div>

        <!-- Equipments -->
        <div class="col-md-10" id="items">
            <el-tabs type="card" v-model="tabs" @tab-click="loadCategory">
                <el-tab-pane :name="`${index}`" :key="category.id" v-for="category, index in categories">
                    <span slot="label"><img :src="image_path_by_name('item', category.image)"/></span>

                    <panel class="items">
                        <div class="filter">
                            <input type="text" class="form-control"
                                   placeholder="Search ..."
                                   v-model="category.filters.name"
                                   @input="filterItemsFromCategory(category)">

                            <div class="filter-properties" v-if="hasPropertyFilter(category)">
                                <div class="line">
                                    <el-checkbox @change="filterItemsFromCategoryByProperty(category)"
                                                 v-model="category.filters.energy">Energy</el-checkbox>
                                    <el-checkbox @change="filterItemsFromCategoryByProperty(category)"
                                                 v-model="category.filters.death">Death</el-checkbox>
                                    <el-checkbox @change="filterItemsFromCategoryByProperty(category)"
                                                 v-model="category.filters.earth">Earth</el-checkbox>
                                    <el-checkbox @change="filterItemsFromCategoryByProperty(category)"
                                                 v-model="category.filters.fire">Fire</el-checkbox>
                                    <el-checkbox @change="filterItemsFromCategoryByProperty(category)"
                                                 v-model="category.filters.ice">Ice</el-checkbox>
                                    <el-checkbox @change="filterItemsFromCategoryByProperty(category)"
                                                 v-model="category.filters['protection physical']">Physical</el-checkbox>
                                    <el-checkbox @change="filterItemsFromCategoryByProperty(category)"
                                                 v-model="category.filters.speed">Speed</el-checkbox>
                                    <el-checkbox @change="filterItemsFromCategoryByProperty(category)"
                                                 v-model="category.filters['magic level']">Magic Level</el-checkbox>
                                </div>
                            </div>
                        </div>

                        <page-load class="no-padding row" :loading="category.loading">
                            <item :item.sync="item" :category="category" :key="item.id"
                                  v-for="item in category.items"
                                  @equiped="equipItem"/>
                        </page-load>
                    </panel>
                </el-tab-pane>
            </el-tabs>
        </div>
    </div>
</template>

<script>
    import SlotItem from './SlotItem'
    import Item from './Item'
    import services from '../services'
    import { isEmpty, filter } from 'lodash'

    export default {
        props: ['slots'],

        components: { SlotItem, Item },

        data () {
            return {
                tabs: '0',
                categories: []
            }
        },

        computed: {
            categoriesId () {
                return this.categories.map(category => category.id)
            }
        },

        methods: {
            loadCategories () {
                services.getCategories()
                    .then(response => {
                        this.categories = response.data.map(category => {
                            return {
                                ...category,
                                items: [],
                                loading: true,
                                filters: {
                                    name: '',
                                    energy: false,
                                    death: false,
                                    earth: false,
                                    fire: false,
                                    ice: false,
                                    speed: false,
                                    'protection physical': false,
                                    'magic level': false,
                                }
                            }
                        })
                        this.categories.length ? this.loadItems(this.categories[0]) : null
                    })
            },

            loadItems (category) {
                services.getItems(category.id)
                    .then(response => {
                        const index = this.categoriesId.indexOf(category.id)
                        this.categories[index].items = response.data.map(item => {
                            return { ...item, visible: true, active: false, imbuements: [] }
                        })
                        this.categories[index].loading = false
                    })
                    .catch(() => {
                        const index = this.categoriesId.indexOf(category.id)
                        this.categories[index].loading = false
                    })
            },

            loadCategory (tab) {
                const index = tab.index
                const category = this.categories[index]
                const items = category.items

                ! items.length ? this.loadItems(category) : null
            },

            filterItemsFromCategory (category) {
                const filter = category.filters.name

                category.items.forEach(item => {
                    item.visible = item.title.toLowerCase().includes(filter)
                })
            },

            filterItemsFromCategoryByProperty (category) {
                const properties = Object.keys(category.filters).filter(property => category.filters[property])

                category.items.forEach(item => {
                    const props = item.properties.filter(property => properties.indexOf(property.property) !== - 1)
                    item.visible = properties.length ? !! props.length : true
                })
            },

            equipItem (response) {
                const category = this.getCategoryById(response.category).category
                const itemIndex = category.items.map(item => item.id).indexOf(response.item)
                const item = category.items[itemIndex]

                this.validate(category, item)

                this.categories[this.getCategoryById(response.category).index].items.forEach(item => item.active = item.id == response.item ? true : false)
                this.slots[response.slot] = item
            },

            getCategoryById (id) {
                const index = this.categoriesId.indexOf(id)
                return { category: this.categories[index], index }
            },

            validate (category, item) {
                const properties = item.properties
                const twoHanded = properties.map(property => property.property).indexOf('two-handed')
                if (twoHanded !== - 1 && properties[twoHanded] && category.slot == 'weapon') {
                    this.slots['shield'] = []
                    this.categories[this.getCategoryById(41).index].items.forEach(item => item.active = false)
                    this.categories[this.getCategoryById(42).index].items.forEach(item => item.active = false)
                }

                if (category.slot == 'shield'
                    && ! isEmpty(this.slots['weapon'])
                    && this.slots['weapon'].properties.map(property => property.property).indexOf('two-handed') !== - 1) {

                    this.slots['weapon'] = []
                    this.categories[this.getCategoryById(5).index].items.forEach(item => item.active = false)
                    this.categories[this.getCategoryById(10).index].items.forEach(item => item.active = false)
                    this.categories[this.getCategoryById(16).index].items.forEach(item => item.active = false)
                    this.categories[this.getCategoryById(39).index].items.forEach(item => item.active = false)
                    this.categories[this.getCategoryById(44).index].items.forEach(item => item.active = false)
                    this.categories[this.getCategoryById(49).index].items.forEach(item => item.active = false)
                }
            },

            removeSlot (slot) {
                this.slots[slot] = {}
            },

            hasPropertyFilter (category) {
                switch (category.slot) {
                    case 'weapon':
                    case 'backpack':
                    case 'ammunition':
                        return false
                    default:
                        return true
                }
            }
        },

        mounted () {
            this.loadCategories()
        }
    }
</script>