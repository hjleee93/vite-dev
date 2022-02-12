<template>
  <div class="map-overlay top">
        <div class="map-overlay-inner">
            <label>x point : {{ xPoint }}</label>
            <label>y point : {{ yPoint }}</label>
            <label>lat : {{ lat }}</label>
            <label>lng : {{ lng }}</label>
            <button @click="$emit('onSelectedReset')">clear tiles</button>
        </div>
    </div>
</template>

<script>
export default {
    nema: 'mapInfo',

}
</script>

<style>

</style>