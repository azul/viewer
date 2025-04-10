<!--
 - @copyright Copyright (c) 2019 John Molakvoæ <skjnldsv@protonmail.com>
 - @copyright Copyright (c) 2020 Gary Kim <gary@garykim.dev>
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
	<Modal
		v-if="initiated || currentFile.modal"
		id="viewer-content"
		:class="{'icon-loading': !currentFile.loaded && !currentFile.failed}"
		:clear-view-delay="isTesting ? -1 : 5000 /* prevent cypress timeouts */"
		:dark="true"
		:enable-slideshow="hasPrevious || hasNext"
		:enable-swipe="canSwipe"
		:has-next="hasNext"
		:has-previous="hasPrevious"
		:size="isMobile ? 'full' : 'large'"
		:spread-navigation="true"
		:style="{width: isSidebarShown ? `calc(100% - ${sidebarWidth}px)` : null}"
		:title="currentFile.basename"
		:view="currentFile.modal"
		@close="close"
		@previous="previous"
		@next="next">
		<!-- ACTIONS -->
		<template #actions>
			<ActionButton
				v-if="Sidebar && !isSidebarShown"
				icon="icon-menu-sidebar-white-forced"
				@click="showSidebar">
				{{ t('viewer', 'Open sidebar') }}
			</ActionButton>
		</template>

		<!-- PREVIOUS -->
		<component
			:is="previousFile.modal"
			v-if="previousFile && !previousFile.failed"
			:key="previousFile.fileid"
			ref="previous-content"
			v-bind="previousFile"
			:file-list="fileList"
			class="hidden-visually file-view"
			@error="previousFailed" />
		<Error
			v-else-if="previousFile"
			class="hidden-visually"
			:name="previousFile.basename" />

		<!-- CURRENT -->
		<component
			:is="currentFile.modal"
			v-if="!currentFile.failed"
			:key="currentFile.fileid"
			ref="content"
			:active="true"
			:can-swipe.sync="canSwipe"
			v-bind="currentFile"
			:file-list="fileList"
			:is-full-screen="isFullscreen"
			:loaded.sync="currentFile.loaded"
			:is-sidebar-shown="isSidebarShown"
			class="file-view active"
			@error="currentFailed" />
		<Error
			v-else
			:name="currentFile.basename" />

		<!-- NEXT -->
		<component
			:is="nextFile.modal"
			v-if="nextFile && !nextFile.failed"
			:key="nextFile.fileid"
			ref="next-content"
			v-bind="nextFile"
			:file-list="fileList"
			class="hidden-visually file-view"
			@error="nextFailed" />
		<Error
			v-else-if="nextFile"
			class="hidden-visually"
			:name="nextFile.basename" />
	</Modal>
</template>

<script>
import { getRootPath } from '../utils/davUtils'
import ActionButton from '@nextcloud/vue/dist/Components/ActionButton'
import isFullscreen from '@nextcloud/vue/dist/Mixins/isFullscreen'
import isMobile from '@nextcloud/vue/dist/Mixins/isMobile'
import Modal from '@nextcloud/vue/dist/Components/Modal'
import Vue from 'vue'

import { extractFilePaths, sortCompare } from '../utils/fileUtils'
import Error from '../components/Error'
import File from '../models/file'
import Mime from '../mixins/Mime'
import getFileList from '../services/FileList'
import getFileInfo from '../services/FileInfo'

import cancelableRequest from '../utils/CancelableRequest'

export default {
	name: 'Viewer',

	components: {
		ActionButton,
		Modal,
		Error,
	},

	mixins: [isMobile, isFullscreen],

	data: () => ({
		// Reactivity bindings
		Viewer: OCA.Viewer.state,
		Sidebar: null,
		handlers: OCA.Viewer.availableHandlers,

		// Viewer variables
		components: {},
		mimeGroups: {},
		registeredHandlers: [],

		// Files variables
		currentIndex: 0,
		previousFile: {},
		currentFile: {},
		nextFile: {},
		fileList: [],

		// States
		isLoaded: false,
		initiated: false,

		// cancellable requests
		cancelRequestFile: () => {},
		cancelRequestFolder: () => {},

		// Flags
		sidebarWidth: 0,
		isSidebarShown: false,
		canSwipe: true,
		isStandalone: !(OCA && OCA.Files && 'fileActions' in OCA.Files),
		isTesting,

		root: getRootPath(),
	}),

	computed: {
		hasPrevious() {
			return this.fileList.length > 1
		},
		hasNext() {
			return this.fileList.length > 1
		},
		file() {
			return this.Viewer.file
		},
		files() {
			return this.Viewer.files
		},
		loadMore() {
			return this.Viewer.loadMore
		},
		isEndOfList() {
			return this.currentIndex === this.fileList.length - 1
		},
		/**
		 * Returns the path to the current opened file in the sidebar.
		 *
		 * If the sidebar is available but closed an empty string is returned.
		 * If the sidebar is not available null is returned.
		 *
		 * @returns {string|null} the path to the current opened file in the
		 *          sidebar, if any.
		 */
		sidebarFile() {
			return this.Sidebar && this.Sidebar.file
		},
	},

	watch: {
		// make sure any late external app can register handlers
		// should not happens if external apps do not wait for
		// the DOMContentLoaded event!
		handlers: function() {
			// make sure the viewer is done registering handlers
			// so we only register handlers added AFTER the init
			// of the viewer
			if (this.isLoaded) {
				console.error('Please do NOT wait for the DOMContentLoaded before registering your viewer handler')
				const handler = this.handlers[this.handlers.length - 1]
				// register all primary components mimes
				this.registerHandler(handler)
				// then register aliases. We need to have the components
				// first so we can bind the alias to them.
				this.registerHandlerAlias(handler)
			}
		},

		file: function(path) {
			// we got a valid path! Load file...
			if (path.trim() !== '') {
				console.info('Opening viewer for file ', path)
				this.openFile(path)
			} else {
				// path is empty, closing!
				this.close()
			}
		},

		files: function(fileList) {
			// the files list changed, let's update the current opened index
			const currentIndex = fileList.findIndex(file => file.basename === this.currentFile.basename)
			if (currentIndex > -1) {
				this.currentIndex = currentIndex
				console.debug('The files list changed, new current file index is', currentIndex)
			}
			// finally replace the fileList
			this.fileList = fileList
		},

		// user reached the end of list
		isEndOfList: async function(isEndOfList) {
			if (!isEndOfList) {
				return
			}

			// if we have a loadMore handler, let's fetch more files
			if (this.loadMore && typeof this.loadMore === 'function') {
				console.debug('Fetching additional files...')
				const list = await this.loadMore()

				if (Array.isArray(list) && list.length > 0) {
					this.fileList.push(...list)
				}
			}
		},

		sidebarFile: function(file) {
			// TODO: implement sidebar event bus
			if (file === '') {
				this.hideAppsSidebar()
			}
		},
	},

	beforeMount() {
		// register on load
		document.addEventListener('DOMContentLoaded', event => {
			// register all primary components mimes
			this.handlers.forEach(handler => {
				this.registerHandler(handler)
			})

			// then register aliases. We need to have the components
			// first so we can bind the alias to them.
			this.handlers.forEach(handler => {
				this.registerHandlerAlias(handler)
			})
			this.isLoaded = true

			// bind Sidebar if available
			if (OCA.Files && OCA.Files.Sidebar) {
				this.Sidebar = OCA.Files.Sidebar.state
			}
		})

		window.addEventListener('resize', this.onResize)

		if (this.isStandalone) {
			console.debug('No OCA.Files app found, viewer is now in standalone mode')
		}
	},

	beforeDestroy() {
		window.removeEventListener('resize', this.onResize)
	},

	methods: {
		/**
		 * Open the view and display the clicked file
		 *
		 * @param {string} path the file path to open
		 */
		async openFile(path) {
			// cancel any previous requests
			this.cancelRequestFile()
			this.cancelRequestFolder()

			// do not open the same file again
			if (path === this.currentFile.path) {
				return
			}

			// initial loading start
			this.initiated = true
			const { request: fileRequest, cancel: cancelRequestFile } = cancelableRequest(getFileInfo)
			const { request: folderRequest, cancel: cancelRequestFolder } = cancelableRequest(getFileList)
			this.cancelRequestFile = cancelRequestFile
			this.cancelRequestFolder = cancelRequestFolder

			// extcrat needed info from path
			const [dirPath, fileName] = extractFilePaths(path)

			// prevent scrolling while opened
			document.body.style.overflow = 'hidden'

			// swap title with original one
			const title = document.getElementsByTagName('head')[0].getElementsByTagName('title')[0]
			if (title && !title.dataset.old) {
				title.dataset.old = document.title
				document.title = `${fileName} - ${OC.theme.title}`
			}

			try {

				// retrieve, sort and store file List
				let fileInfo = await fileRequest(path)

				// get original mime
				let mime = fileInfo.mime

				// check if part of a group, if so retrieve full files list
				const group = this.mimeGroups[mime]
				if (this.files && this.files.length > 0) {
					console.debug('A files list have been provided. No folder content will be fetched.')
					// we won't sort files here, let's use the order the array has
					this.fileList = this.files

					// store current position
					this.currentIndex = this.fileList.findIndex(file => file.basename === fileName)
				} else if (group) {
					const mimes = this.mimeGroups[group]
						? this.mimeGroups[group]
						: [mime]

					// retrieve folder list
					const fileList = await folderRequest(dirPath)

					// filter out the unwanted mimes
					const filteredFiles = fileList.filter(file => file.mime && mimes.indexOf(file.mime) !== -1)

					// sort like the files list
					// TODO: implement global sorting API
					// https://github.com/nextcloud/server/blob/a83b79c5f8ab20ed9b4d751167417a65fa3c42b8/apps/files/lib/Controller/ApiController.php#L247
					this.fileList = filteredFiles.sort((a, b) => sortCompare(a, b, 'basename'))

					// store current position
					this.currentIndex = this.fileList.findIndex(file => file.basename === fileName)
				} else {
					this.currentIndex = 0
					this.fileList = [fileInfo]
				}

				// get saved fileInfo
				fileInfo = this.fileList[this.currentIndex]

				// override mimetype if existing alias
				if (!this.components[mime]) {
					mime = mime.split('/')[0]
				}

				// if we have a valid mime, show it!
				if (this.components[mime]) {
					this.currentFile = new File(fileInfo, mime, this.components[mime])
					this.updatePreviousNext()
				} else {
					console.error(`The following file could not be displayed`, fileName, fileInfo)
					this.close()
				}

				// if sidebar was opened before, let's update the file
				this.changeSidebar()
			} catch (error) {
				console.error(error)
			}
		},

		/**
		 * Open the view and display the file from the file list
		 *
		 * @param {Object} fileInfo the opened file info
		 */
		openFileFromList(fileInfo) {
			// override mimetype if existing alias
			const mime = fileInfo.mime
			this.currentFile = new File(fileInfo, mime, this.components[mime])
			this.changeSidebar()
			this.updatePreviousNext()
		},

		/**
		 * Show sidebar if available and a file is already opened
		 */
		changeSidebar() {
			if (this.sidebarFile) {
				this.showSidebar()
			}
		},

		/**
		 * Update the previous and next file components
		 */
		updatePreviousNext() {
			const prev = this.fileList[this.currentIndex - 1]
			const next = this.fileList[this.currentIndex + 1]

			if (prev) {
				const mime = prev.mime
				if (this.components[mime]) {
					this.previousFile = new File(prev, mime, this.components[mime])
				}
			} else {
				// RESET
				this.previousFile = null
			}

			if (next) {
				const mime = next.mime
				if (this.components[mime]) {
					this.nextFile = new File(next, mime, this.components[mime])
				}
			} else {
				// RESET
				this.nextFile = null
			}

		},

		/**
		 * Registering possible new handers
		 *
		 * @param {Object} handler the handler to register
		 * @param {String} handler.id unique handler identifier
		 * @param {Array} handler.mimes list of valid mimes compatible with the handler
		 * @param {Object} handler.component a vuejs component to render when a file matching the mime list is opened
		 * @param {String} [handler.group] a group name to be associated with for the slideshow
		 */
		registerHandler(handler) {
			// checking if handler is not already registered
			if (handler.id && this.registeredHandlers.indexOf(handler.id) > -1) {
				console.error(`The following handler is already registered`, handler)
				return
			}

			// checking valid handler id
			if (!handler.id || handler.id.trim() === '' || typeof handler.id !== 'string') {
				console.error(`The following handler doesn't have a valid id`, handler)
				return
			}

			// checking if no valid mimes data but alias. If so, skipping...
			if (!(handler.mimes && Array.isArray(handler.mimes)) && handler.mimesAliases) {
				return
			}

			// Nothing available to process! Failure
			if (!(handler.mimes && Array.isArray(handler.mimes)) && !handler.mimesAliases) {
				console.error(`The following handler doesn't have a valid mime array`, handler)
				return
			}

			// checking valid handler component data
			if ((!handler.component || typeof handler.component !== 'object')) {
				console.error(`The following handler doesn't have a valid component`, handler)
				return
			}

			// force apply mixin
			handler.component.mixins = [...handler.component.mixins || [], Mime]

			// parsing mimes registration
			if (handler.mimes) {
				handler.mimes.forEach(mime => {
					// checking valid mime
					if (this.components[mime]) {
						console.error(`The following mime is already registered`, mime, handler)
						return
					}

					// register file action and groups
					this.registerAction({ mime, group: handler.group })

					// register mime's component
					this.components[mime] = handler.component
					Vue.component(handler.component.name, handler.component)

					// set the handler as registered
					this.registeredHandlers.push(handler.id)
				})
			}
		},

		registerHandlerAlias(handler) {
			// parsing aliases registration
			if (handler.mimesAliases) {
				Object.keys(handler.mimesAliases).forEach(mime => {

					if (handler.mimesAliases && typeof handler.mimesAliases !== 'object') {
						console.error(`The following handler doesn't have a valid mimesAliases object`, handler)
						return

					}

					// this is the targeted alias
					const alias = handler.mimesAliases[mime]

					// checking valid mime
					if (this.components[mime]) {
						console.error(`The following mime is already registered`, mime, handler)
						return
					}
					if (!this.components[alias]) {
						console.error(`The requested alias does not exists`, alias, mime, handler)
						return
					}

					// register file action and groups if the request alias had a group
					this.registerAction({ mime, group: this.mimeGroups[alias] })

					// register mime's component
					this.components[mime] = this.components[alias]

					// set the handler as registered
					this.registeredHandlers.push(handler.id)
				})
			}
		},

		registerAction({ mime, group }) {
			if (!this.isStandalone) {
				// unregistered handler, let's go!
				OCA.Files.fileActions.registerAction({
					name: 'view',
					displayName: t('viewer', 'View'),
					mime: mime,
					permissions: OC.PERMISSION_READ,
					actionHandler: (name, { dir }) => {
						// replace potential leading double slashes
						OCA.Viewer.open(`${dir}/${name}`.replace(/^\/\//, '/'))
					},
				})
				OCA.Files.fileActions.setDefault(mime, 'view')
			}

			// register groups
			if (group) {
				this.mimeGroups[mime] = group
				// init if undefined
				if (!this.mimeGroups[group]) {
					this.mimeGroups[group] = []
				}
				this.mimeGroups[group].push(mime)
			}
		},

		/**
		 * Close the viewer
		 */
		close() {
			// reset all properties
			OCA.Viewer.close()
			this.currentFile = {}
			this.currentModal = null
			this.fileList = []
			this.initiated = false
			this.hideAppsSidebar()

			// cancel requests
			this.cancelRequestFile()
			this.cancelRequestFolder()

			// restore default
			document.body.style.overflow = null

			// swap back original title
			const title = document.getElementsByTagName('head')[0].getElementsByTagName('title')[0]
			if (title && title.dataset.old) {
				document.title = title.dataset.old
				delete title.dataset.old
			}
		},

		/**
		 * Open previous available file
		 */
		previous() {
			this.currentIndex--
			if (this.currentIndex < 0) {
				this.currentIndex = this.fileList.length - 1
			}

			this.openFileFromList(this.fileList[this.currentIndex])
		},

		/**
		 * Open next available file
		 */
		next() {
			this.currentIndex++
			if (this.currentIndex > this.fileList.length - 1) {
				this.currentIndex = 0
			}

			this.openFileFromList(this.fileList[this.currentIndex])
		},

		/**
		 * Failures handlers
		 */
		previousFailed() {
			this.previousFile.failed = true
		},

		currentFailed() {
			this.currentFile.failed = true
		},

		nextFailed() {
			this.nextFile.failed = true
		},

		/**
		 * Show the sharing sidebar
		 */

		async showSidebar() {
			// Open the sidebar sharing tab
			await OCA.Files.Sidebar.open(this.currentFile.filename)
			this.showAppsSidebar()
		},

		showAppsSidebar() {
			this.isSidebarShown = true
			const sidebar = document.getElementById('app-sidebar')
			if (sidebar) {
				sidebar.classList.add('app-sidebar--full')
				this.sidebarWidth = sidebar.offsetWidth
			}
		},

		hideAppsSidebar() {
			this.isSidebarShown = false
			const sidebar = document.getElementById('app-sidebar')
			if (sidebar) {
				sidebar.classList.remove('app-sidebar--full')
			}
		},

		onResize(event) {
			// update sidebar width
			const sidebar = document.getElementById('app-sidebar')
			if (sidebar) {
				this.sidebarWidth = sidebar.offsetWidth
			}
		},
	},
}
</script>

<style lang="scss">
#viewer-content.modal-mask {
	transition: width ease 100ms;
	.modal-container {
		display: flex !important;
		width: auto !important;
		border-radius: 0 !important;
		// let the mime components manage their own background-color
		background-color: transparent;
		justify-content: center;
		align-items: center;
	}
}

.component-fade-enter-active, .component-fade-leave-active {
	transition: opacity .3s ease;
}

.component-fade-enter, .component-fade-leave-to {
	opacity: 0;
}

// force white icon
.icon-menu-sidebar-white-forced {
	background-image: url('~Assets/menu-sidebar-white.svg');
}

.file-view {
	transition: height 100ms ease,
		width 100ms ease;
}

// Override vue componets scss
#app-sidebar.app-sidebar--full {
	position: fixed !important;
	top: 0 !important;
	height: 100% !important;
	z-index: 2025 !important;
	.thumbnailContainer {
		display: none !important;
	}
}

// put autocomplete over full sidebar
// TODO: remove when new sharing sidebar (18)
// is the min-version of viewer
.ui-autocomplete {
	z-index: 2050 !important;
}
</style>
