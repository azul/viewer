<!--
 - @copyright Copyright (c) 2019 John Molakvoæ <skjnldsv@protonmail.com>
 -
 - @author John Molakvoæ <skjnldsv@protonmail.com>
 -
 - @license GNU AGPL version 3 or any later version
 -
 - This program is free software: you can redistribute it and/or modify
 - it under the terms of the GNU Affero General Public License as
 - published by the Free Software Foundation, either version 3 of the
 - License, or (at your option) any later version.
 -
 - This program is distributed in the hope that it will be useful,
 - but WITHOUT ANY WARRANTY; without even the implied warranty of
 - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
 - GNU Affero General Public License for more details.
 -
 - You should have received a copy of the GNU Affero General Public License
 - along with this program. If not, see <http://www.gnu.org/licenses/>.
 -
 -->

<template>
	<img
		:class="{
			dragging,
			loaded,
			zoomed: zoomRatio !== 1
		}"
		:src="data"
		:style="{
			height: minHeight,
			width: minWidth,
			marginTop: shiftY + 'px',
			marginLeft: shiftX + 'px'
		}"
		@load="updateImgSize"
		@wheel="updateZoom"
		@dblclick.prevent="onDblclick"
		@mousedown.prevent="dragStart">
</template>

<script>
import axios from '@nextcloud/axios'
import Vue from 'vue'
import AsyncComputed from 'vue-async-computed'

Vue.use(AsyncComputed)

export default {
	name: 'Images',

	props: {
		// file etag, used for cache reset
		etag: {
			type: String,
			required: true,
		},
	},
	data() {
		return {
			dragging: false,
			shiftX: 0,
			shiftY: 0,
			zoomRatio: 1,
		}
	},
	computed: {
		zoomHeight() {
			return Math.round(this.height * this.zoomRatio)
		},
		zoomWidth() {
			return Math.round(this.width * this.zoomRatio)
		},

		// prevent images smaller than 100px
		minHeight() {
			return this.zoomWidth < 100
				? null
				: this.zoomHeight + 'px'
		},
		// prevent images smaller than 100px
		minWidth() {
			return this.zoomHeight < 100
				? null
				: this.zoomWidth + 'px'
		},
	},

	asyncComputed: {
		data() {
			switch (this.mime) {
			case 'image/svg+xml':
				return this.getBase64FromImage()
			case 'image/gif':
				return this.davPath
			default:
				return this.previewpath
			}
		},
	},
	watch: {
		active: function(val, old) {
			// the item was hidden before and is now the current view
			if (val === true && old === false) {
				this.resetZoom()
				// end the dragging if your mouse go out of the content
				window.addEventListener('mouseout', this.dragEnd)
			// the item is not displayed
			} else if (val === false) {
				window.removeEventListener('mouseout', this.dragEnd)
			}
		},
	},
	methods: {
		// Updates the dimensions of the modal
		updateImgSize() {
			this.naturalHeight = this.$el.naturalHeight
			this.naturalWidth = this.$el.naturalWidth

			this.updateHeightWidth()
			this.doneLoading()
		},

		/**
		 * Manually retrieve the path and return its base64
		 *
		 * @returns {String}
		 */
		async getBase64FromImage() {
			const file = await axios.get(this.davPath)
			return `data:${this.mime};base64,${btoa(file.data)}`
		},

		/**
		 * Handle zooming
		 *
		 * @param {Event} event the scroll event
		 * @returns {null}
		 */
		updateZoom(event) {
			event.stopPropagation()
			event.preventDefault()

			// scrolling position relative to the image
			const scrollX = event.clientX - this.$el.x - (this.width * this.zoomRatio / 2)
			const scrollY = event.clientY - this.$el.y - (this.height * this.zoomRatio / 2)
			const scrollPercX = Math.round(scrollX / (this.width * this.zoomRatio) * 100) / 100
			const scrollPercY = Math.round(scrollY / (this.height * this.zoomRatio) * 100) / 100
			const isZoomIn = event.deltaY < 0

			const newZoomRatio = isZoomIn
				? Math.min(this.zoomRatio + 0.1, 5) // prevent too big zoom
				: Math.max(this.zoomRatio - 0.1, 1) // prevent too small zoom

			// do not continue, img is back to its original state
			if (newZoomRatio === 1) {
				return this.resetZoom()
			}

			// calc how much the img grow from its current size
			// and adjust the margin accordingly
			const growX = this.width * newZoomRatio - this.width * this.zoomRatio
			const growY = this.height * newZoomRatio - this.height * this.zoomRatio

			// compensate for existing margins
			this.disableSwipe()
			this.shiftX = this.shiftX + Math.round(-scrollPercX * growX)
			this.shiftY = this.shiftY + Math.round(-scrollPercY * growY)
			this.zoomRatio = newZoomRatio
		},

		resetZoom() {
			this.enableSwipe()
			this.zoomRatio = 1
			this.shiftX = 0
			this.shiftY = 0
		},

		/**
		 * Dragging handlers
		 *
		 * @param {Event} event the event
		 */
		dragStart(event) {
			const { pageX, pageY } = event

			this.dragX = pageX
			this.dragY = pageY
			this.dragging = true
			this.$el.onmouseup = this.dragEnd
			this.$el.onmousemove = this.dragHandler
		},
		dragEnd(event) {
			event.preventDefault()

			this.dragging = false
			this.$el.onmouseup = null
			this.$el.onmousemove = null
		},
		dragHandler(event) {
			event.preventDefault()
			const { pageX, pageY } = event

			if (this.dragging && this.zoomRatio > 1 && pageX > 0 && pageY > 0) {
				const moveX = this.shiftX + (pageX - this.dragX)
				const moveY = this.shiftY + (pageY - this.dragY)
				const growX = this.zoomWidth - this.width
				const growY = this.zoomHeight - this.height

				this.shiftX = Math.min(Math.max(moveX, -growX / 2), growX / 2)
				this.shiftY = Math.min(Math.max(moveY, -growY / 2), growX / 2)
				this.dragX = pageX
				this.dragY = pageY
			}
		},
		onDblclick() {
			if (this.zoomRatio > 1) {
				this.resetZoom()
			} else {
				this.zoomRatio = 1.3
			}
		},
	},
}
</script>

<style scoped lang="scss">
$checkered-size: 8px;
$checkered-color: #efefef;

img {
	max-width: 100%;
	max-height: 100%;
	align-self: center;
	justify-self: center;
	// black while loading
	background-color: #000;
	// animate zooming/resize
	transition: height 100ms ease,
		width 100ms ease,
		margin-top 100ms ease,
		margin-left 100ms ease;
	// show checkered bg on hover if not currently zooming (but ok if zoomed)
	&:hover {
		background-image: linear-gradient(45deg, #{$checkered-color} 25%, transparent 25%),
			linear-gradient(45deg, transparent 75%, #{$checkered-color} 75%),
			linear-gradient(45deg, transparent 75%, #{$checkered-color} 75%),
			linear-gradient(45deg, #{$checkered-color} 25%, #fff 25%);
		background-size: 2 * $checkered-size 2 * $checkered-size;
		background-position: 0 0, 0 0, -#{$checkered-size} -#{$checkered-size}, $checkered-size $checkered-size;
	}
	&.loaded {
		// white once done loading
		background-color: #fff;
	}
	&.zoomed {
		position: absolute;
		max-height: none;
		max-width: none;
		z-index: 10010;
		cursor: move;
	}

	&.dragging {
		transition: none !important;
		cursor: move;
	}
}
</style>
