<template>
    <form @submit.prevent="getOrder()">
        <input v-model="id" type="text" />
    </form>
</template>

<script>
export default {
    data() {
        return {
            id: null,
        }
    },

    methods: {
        getOrder() {
            if (this.id > 0) {
                this.$http
                    .get(`http://192.168.1.12:8080/api/labeling/jobHeaderItems.php?job_header_id=${this.id}`)
                    .then(response => {
                        this.$emit('order-updated', response.data)
                    })
                    .catch(error => console.log(error));
            }
        }
    },
}
</script>