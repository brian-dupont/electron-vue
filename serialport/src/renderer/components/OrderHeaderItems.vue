<template>
    <div v-if="this.orderItems.length > 0">
        <ol>
            <li 
                v-for="(item, index) in this.orderItems" 
                :key="index"
                @click="getItemDetails"
                :data-job-item-id="item.job_item_id"
            >
                {{ item.item_number }} - {{ item.order_quantity }}
            </li>
        </ol>
    </div>
</template>

<script>
export default {
    props: {
        orderItems: {
            type: Array,
        }
    },

    data() {
        return {
            items: this.orderItems,
        }
    },

    methods: {
        getItemDetails(event) {
            let jobItemId = event.target.getAttribute('data-job-item-id')

            if (jobItemId > 0) {
                this.$http
                    .get(`http://192.168.1.12:8080/api/labeling/jobItemDetails.php?job_item_id=${jobItemId}`)
                    .then(response => {
                        console.log(response.data)
                        this.$emit('item-updated', response.data)
                    })
                    .catch(error => console.log(error));
            }
        }
    },
}
</script>

<style>
li:hover {
    cursor: pointer;
}
</style>