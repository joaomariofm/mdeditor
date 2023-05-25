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
	const target = event.target as HTMLElement
	const { newContent, matches } = checkForMdSyntax(target.innerHTML)

	const { startCharacterIndex, cursorStartNodePath } =  getCursorPosition()

  const parsedContent = parseContent(newContent)
  content.value = parsedContent

	nextTick(() => {
		setCursor(startCharacterIndex, cursorStartNodePath, matches)
	})
}

function checkForMdSyntax(newContent: string) {
	const strongRegex = /\*{2}.*?\*{2}/g
	const strongMatches = newContent.match(strongRegex)

	if (strongMatches) {
		strongMatches.forEach(match => {
			const newMatch = match.replace(/\*/g, '')
			newContent = newContent.replace(match, `<strong>${newMatch}</strong>`)
		})
	}

	return { 
		newContent,
		matches: strongMatches ? true : false,
	}
}

function parseContent(newContent: string): string {
  const parser = new DOMParser()
  const parsedDocument = parser.parseFromString(newContent, 'text/html')
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
		cursorStartNodePath: number[],
		matches: boolean
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

	if (matches) {
		if (testingStartNode?.parentNode.childNodes[testingStartNode?.parentNode.childNodes.length - 1].nodeName === 'BR') {
			testingStartNode = testingStartNode?.parentNode.childNodes[testingStartNode?.parentNode.childNodes.length - 1]
		} else {
			const br = document.createElement('br')
			testingStartNode?.parentNode?.insertBefore(br, testingStartNode?.nextSibling)
			testingStartNode = testingStartNode?.parentNode.childNodes[testingStartNode?.parentNode.childNodes.length - 1]
		}

		startCharacterIndex = 0
	}

	range.setStart(testingStartNode, startCharacterIndex)

	selection?.removeAllRanges()
	selection?.addRange(range)
}
</script>

<style>
#editable {
	font-family: 'Roboto', sans-serif;
	color: #f8fafc;

	min-height: 100vh;
	width: 50vw;

	box-sizing: border-box;
	padding: 3rem 1rem 0 1rem;
}

#editable:focus {
	outline: none;
}
	

[contenteditable=true]:empty:before{
  content: attr(placeholder);
	color: #3f3f46;
	font-weight: bold;
  pointer-events: none;
  display: block; /* For Firefox */
}
</style>
