<template>
    <div v-html="require(`./../../../svg/${icon}.svg`)"></div>
</template>

<style lang="scss" module>
.svg {
    fill: currentColor;
    height: 1em;
    margin-top: -4px;
    vertical-align: middle;
    width: 1em;

    g *, path {
        fill: currentColor;
    }
}
</style>

<script>
export default {
    name: 'svg-icon',
    props: {
        icon: {
            type: String,
            default: ''
        }
    },
    mounted() {
        this.$el.firstChild.classList.add(this.$style.svg);
        this.$el.firstChild.removeAttribute('height');
        this.$el.firstChild.removeAttribute('width');
    },
    watch: {
        icon() {
            this.$nextTick(() => {
                this.$el.firstChild.classList.add(this.$style.svg);
                // this.$el.querySelector('svg').classList.add(this.$style.svg);
                this.$el.firstChild.removeAttribute('height');
                this.$el.firstChild.removeAttribute('width');
            })
        }
    }
}
</script>

<style lang="scss" scoped>
div {
    display: inline-block;
}
</style>