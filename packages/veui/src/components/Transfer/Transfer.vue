<script>
import CandidatePanel from './_CandidatePanel'
import SelectedPanel from './_SelectedPanel'
import { includes, isString, clone, difference, omit, remove } from 'lodash'
import prefix from '../../mixins/prefix'
import ui from '../../mixins/ui'
import input from '../../mixins/input'
import { focusIn } from '../../utils/dom'

function defaultFilter (type, keyword, item, datasource) {
  return includes(item.label, keyword)
}

export default {
  name: 'veui-transfer',
  mixins: [prefix, ui, input],
  model: {
    prop: 'selected',
    event: 'select'
  },
  props: {
    datasource: {
      type: Array,
      default () {
        return []
      }
    },
    searchable: {
      type: Boolean,
      default: true
    },
    filter: {
      type: Function,
      default: defaultFilter
    },
    selected: {
      type: Array,
      default () {
        return []
      }
    },
    candidatePlaceholder: String,
    selectedPlaceholder: String,
    selectedShowMode: {
      type: String,
      default: 'tree',
      validator (value) {
        return includes(['tree', 'flat'], value)
      }
    },
    keys: {
      type: [String, Function],
      default: 'value'
    }
  },
  data () {
    return {
      localSelected: this.selected ? clone(this.selected) : []
    }
  },
  computed: {
    isSelectable () {
      return !this.realDisabled && !this.realReadonly
    },
    realKeys () {
      if (isString(this.keys)) {
        return source => source[this.keys]
      }

      return this.keys
    },
    selectedItems () {
      let selected = clone(this.localSelected)
      let walk = datasource => {
        let res = []
        datasource.forEach(source => {
          if (source.children && source.children.length) {
            let selectedChildren = walk(source.children)
            if (selectedChildren.length) {
              let item = omit(source, 'children')
              item.children = selectedChildren
              res.push(item)
            }
          } else if (
            remove(selected, value => value === source.value).length !== 0
          ) {
            res.push(source)
          }
        })
        return res
      }

      return walk(this.datasource)
    }
  },
  watch: {
    selected (v) {
      this.localSelected = v ? clone(v) : []
    }
  },
  methods: {
    handleSelect (val) {
      this.localSelected = val
      this.$emit('select', this.localSelected)
    },

    collectLeafValue (datasource) {
      let value = []

      let walk = data => {
        data.forEach(item => {
          if (item.children && item.children.length && !item.disabled) {
            walk(item.children)
          } else if (!item.disabled) {
            value.push(item.value)
          }
        })
      }
      walk(datasource)
      return value
    },

    focus () {
      focusIn(this.$el)
    }
  },
  render (h) {
    function generateHead (type) {
      let headSlotName = `${type}-head`
      let head = this.$slots[headSlotName]
        ? h('template', { slot: 'head' }, this.$slots[headSlotName])
        : null

      let titleSlotName = `${type}-title`
      let title =
        !head && this.$slots[titleSlotName]
          ? h('template', { slot: 'title' }, this.$slots[titleSlotName])
          : null

      return [head, title]
    }

    function generateItem (type) {
      let itemSlotName = `${type}-item`
      let item = this.$scopedSlots[itemSlotName]
        ? props => this.$scopedSlots[itemSlotName](props)
        : null

      let itemLabelSlotName = `${type}-item-label`
      let itemLabel =
        !item && this.$scopedSlots[itemLabelSlotName]
          ? props => this.$scopedSlots[itemLabelSlotName](props)
          : props => null

      return {
        item,
        'item-label': itemLabel
      }
    }

    function generateNoData (type) {
      let slotName = `${type}-no-data`
      return this.$slots[slotName]
        ? h('template', { slot: 'no-data' }, this.$slots[slotName])
        : null
    }

    return h(
      'div',
      {
        class: {
          [this.$c('input-invalid')]: this.realInvalid,
          [this.$c('transfer')]: true,
          [this.$c('transfer-disabled')]: !this.isSelectable
        },
        attrs: {
          ui: this.ui
        }
      },
      [
        h(
          CandidatePanel,
          {
            props: {
              datasource: this.datasource,
              searchable: this.searchable,
              filter: this.filter,
              placeholder: this.candidatePlaceholder,
              isSelectable: this.isSelectable,
              selected: this.localSelected,
              uiParts: this.uiParts,
              ui: this.realUi
            },
            on: {
              select: (...args) => {
                this.handleSelect(...args)
              },
              selectall: (...args) => {
                this.localSelected = this.collectLeafValue(this.datasource)
                this.$emit('select', this.localSelected)
              }
            },
            scopedSlots: generateItem.call(this, 'candidate'),
            ref: 'candidatePanel'
          },
          [
            ...generateHead.call(this, 'candidate'),
            generateNoData.call(this, 'candidate')
          ]
        ),
        h(
          SelectedPanel,
          {
            props: {
              datasource: this.selectedItems,
              showMode: this.selectedShowMode,
              placeholder: this.selectedPlaceholder,
              isSelectable: this.isSelectable,
              icons: this.icons,
              uiParts: this.uiParts,
              ui: this.realUi
            },
            on: {
              remove: item => {
                this.localSelected = difference(
                  this.localSelected,
                  this.collectLeafValue([item])
                )
                this.$emit('select', this.localSelected)
              },
              removeall: (...args) => {
                this.localSelected = []
                this.$emit('select', this.localSelected)
              }
            },
            scopedSlots: generateItem.call(this, 'selected')
          },
          [
            ...generateHead.call(this, 'selected'),
            generateNoData.call(this, 'selected')
          ]
        )
      ]
    )
  }
}
</script>
