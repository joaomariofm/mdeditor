<template>
	<div id="editable" ref="editable" contenteditable="true" @keyup="remapContent" v-html="content"></div>
</template>

<script setup lang="ts">
import { ref, nextTick } from 'vue'

const content = ref('testing my app')
const editable = ref<HTMLElement | null>(null)

function remapContent(event: Event) {
	const { start, end, startNodePosition, endNodePosition, startNodeParent } =  getCursorPosition()

	const target = event.target as HTMLElement
  const parsedContent = parseContent(target.innerHTML)
  content.value = parsedContent

	nextTick(() => {
		setCursor(start, end, startNodePosition, endNodePosition, startNodeParent)
	})
}

function parseContent(content: string): string {
  const parser = new DOMParser()
  const parsedDocument = parser.parseFromString(content, 'text/html')
  return parsedDocument.body.innerHTML
}

function getCursorPosition() {
	const selection = window.getSelection()
	const range = selection?.getRangeAt(0)
	const start = range?.startOffset
	const end = range?.endOffset
	const startNodeParent = range?.startContainer?.parentNode
	let startNodePosition = 0
	let endNodePosition = 0


	if (startNodeParent?.id === 'editable') {
		startNodePosition = range?.startContainer?.parentNode?.childNodes
			? Array.from(range?.startContainer?.parentNode?.childNodes).indexOf(range?.startContainer)
			: 0
		endNodePosition = range?.endContainer?.parentNode?.childNodes
			? Array.from(range?.endContainer?.parentNode?.childNodes).indexOf(range?.endContainer)
			: 0 
	} else {
		startNodePosition = range?.startContainer?.parentNode?.parentNode?.childNodes
			? Array.from(range?.startContainer?.parentNode?.parentNode?.childNodes).indexOf(range?.startContainer?.parentNode)
			: 0
		endNodePosition = range?.endContainer?.parentNode?.parentNode?.childNodes
			? Array.from(range?.endContainer?.parentNode?.parentNode?.childNodes).indexOf(range?.endContainer?.parentNode)
			: 0
	}

	return {
		start,
		end,
		startNodePosition,
		endNodePosition,
		startNodeParent
	}
}

function setCursor(start: number, end: number, startNodePosition: number, endNodePosition: number, startNodeParent: Node | null) {
	const editor = editable.value

	if (!editor) return

	const range = document.createRange()
	const selection = window.getSelection()
	let startNode = null
	let endNode = null

	if (startNodeParent?.id === 'editable') {
		startNode = editor.childNodes[startNodePosition]
		endNode = editor.childNodes[endNodePosition]
	} else {
		startNode = editor.childNodes[startNodePosition].childNodes[0]
		endNode = editor.childNodes[endNodePosition].childNodes[0]
	}
	
	range.setStart(startNode, start)
	range.setEnd(endNode, end)

	selection?.removeAllRanges()
	selection?.addRange(range)
}
</script>

<style>
#editable {
	min-height: 100vh;

	box-sizing: border-box;
	padding: 1rem 1rem 0 1rem;
}
</style>
