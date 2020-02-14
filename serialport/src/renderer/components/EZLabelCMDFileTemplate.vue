<template>
    <div v-text="this.template"></div>
</template>

<script>
export default {
    props: {
        templateData: {
            default: null,
        },
    },

    data() {
        return {
            
        }
    },

    computed: {
        template() {
            if (this.templateData !== null) {
                return `FORMATNAME=${this.templateData.formatname}
                        formatcount=${this.templateData.formatcount}
                        where=(NUMBER= ${this.templateData.numb_only})
                        SINGLEJOB=ON
                        TESTPRINT=off
                        NUMB="${this.templateData.numb}"
                        LOTN="${this.templateData.lotn}"
                        WT=${this.templateData.wt}
                        BAR=${this.templateData.bar}
                        ALLERGENS="${this.templateData.allergens} "
                        INPUT="${this.templateData.input}"
                        ;`
            }

            return 'No Template'
        }
    }
}
</script>