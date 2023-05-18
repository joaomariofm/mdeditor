<template>
	<div
		id="editable"
		ref="editable"
		@keyup="remapContent"
		v-html="content"
		contenteditable="true"
		spellcheck="false"
		placeholder="Start typing..."
	></div>
</template>

<script setup lang="ts">
import { ref, nextTick } from 'vue'

const content = ref('')
const editable = ref<HTMLElement | null>(null)

interface ParentNodeWithId extends ParentNode {
	id: string;
}

function remapContent(event: Event) {
	const { startCharacterIndex, endCharacterIndex, cursorStartNodePath, cursorEndNodePath } =  getCursorPosition()

	const target = event.target as HTMLElement
  const parsedContent = parseContent(target.innerHTML)
  content.value = parsedContent

	nextTick(() => {
		setCursor(startCharacterIndex, endCharacterIndex, cursorStartNodePath, cursorEndNodePath)
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
	const startCharacterIndex = range?.startOffset ? range?.startOffset : 0
	const endCharacterIndex = range?.endOffset ? range?.endOffset : 0

	const cursorStartNodePath = []
	
	let startingNode = range?.startContainer as ParentNodeWithId | null | undefined
	let startingNodeParent = range?.startContainer?.parentNode as ParentNodeWithId | null | undefined

	while(true) {
		if (startingNodeParent?.id === 'app') break
		cursorStartNodePath.unshift(Array.from(startingNodeParent?.childNodes as NodeListOf<Node>).indexOf(startingNode as Node))

		if (startingNodeParent?.id === 'editable') break
		startingNode = startingNodeParent
		startingNodeParent = startingNodeParent?.parentNode as ParentNodeWithId | null | undefined
	}

	const cursorEndNodePath = []

	let endingNode = range?.endContainer as ParentNodeWithId | null | undefined
	let endingNodeParent = range?.endContainer?.parentNode as ParentNodeWithId | null | undefined

	while(true) {
		if (endingNodeParent?.id === 'app') break
		cursorEndNodePath.unshift(Array.from(endingNodeParent?.childNodes as NodeListOf<Node>).indexOf(endingNode as Node))

		if (endingNodeParent?.id === 'editable') break
		endingNode = endingNodeParent
		endingNodeParent = endingNodeParent?.parentNode as ParentNodeWithId | null | undefined
	}

	return {
		startCharacterIndex,
		endCharacterIndex,
		cursorStartNodePath,
		cursorEndNodePath,
	}
}

function setCursor(
		startCharacterIndex: number,
		endCharacterIndex: number,
		cursorStartNodePath: number[],
		cursorEndNodePath: number[],
	) {
	const editor = editable.value

	if (!editor) return

	const range = document.createRange()
	const selection = window.getSelection()

	let testingStartNode: Node | null | undefined = null

	cursorStartNodePath.forEach((nodeIndex, index) => {
		if (index === 0) {
			testingStartNode = editor.childNodes[nodeIndex]
			return
		}

		testingStartNode = testingStartNode?.childNodes[nodeIndex]
	})

	let testingEndNode: Node | null | undefined = null

	cursorEndNodePath.forEach((nodeIndex, index) => {
		if (index === 0) {
			testingEndNode = editor.childNodes[nodeIndex]
			return
		}

		testingEndNode = testingEndNode?.childNodes[nodeIndex]
	})
	
	range.setStart(testingStartNode, startCharacterIndex)
	range.setEnd(testingEndNode, endCharacterIndex)

	selection?.removeAllRanges()
	selection?.addRange(range)
}
</script>

<style>
#editable {
	font-family: 'Roboto', sans-serif;
	color: #f8fafc;

	min-height: 100vh;

	box-sizing: border-box;
	padding: 1rem 1rem 0 1rem;
}

[contenteditable=true]:empty:before{
  content: attr(placeholder);
	color: #3f3f46;
	font-weight: bold;
  pointer-events: none;
  display: block; /* For Firefox */
}
</style>
