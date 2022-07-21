<script context="module" lang="ts">
	import '../app.css';
	export const prerender = true;
</script>

<script lang="ts">
	import striptags from 'striptags';
	import Span from '../Span.svelte'
	import { titles } from '../titles.ts';

	type Token = {
		text: string,
		trailing: string,
		redacted: boolean,
	}
	type Section = {
		heading: Token,
		paragraphs: Token[][],
	}
	let wikiData;
	let sections = [];
	let wholeText = '';

	let guesses = {};
	let guess = '';

	let solved = false;

	const getArticle = () => {
		const rand = Math.floor(Math.random() * titles.length);
		const title = titles[rand];

		fetch(`https://ko.wikipedia.org/api/rest_v1/page/mobile-sections/${title}`)
		.then(response => response.json())
		.then(data => {
			wikiData = data;
			renderArticle();
		});
	}

	const renderArticle = () => {
		sections = [];
		wholeText = '';
		wikiData.lead.sections.forEach((leadSection, i) => {
			if (i > 100) { return; }

			if (i === 0) {
				const headingTokens = getTokens(wikiData.lead.displaytitle);
				solved = !headingTokens.some(token => token.redacted === true);

				const text = getText(leadSection.text);
				const textParagraphs = getParagraphsWithTokens(text);

				sections.push({ paragraphs: textParagraphs, heading: headingTokens });
			} else {
				sections.push({ paragraphs: [], heading: getTokens(leadSection.line) });
			}
		});

		sections.forEach((section, i) => {
			if (i === 0) { return; }
			const text = getText(wikiData.remaining.sections[i - 1].text);
			const paragraphs = getParagraphsWithTokens(text);
			section.paragraphs = paragraphs;
		});

		sections = sections; // why do i have to do this :(
	}

	const getText = (sectionText: string) => {
		let text = sectionText.replace(/<style.*>.*<\/style>/ig, '')
								.replace(/<table.*>.*<\/table>/ig, '')
		return striptags(text)
			.replace(/&nbsp;/g, ' ')
			.replace(/&(?:amp);/g, '&')
			.replace(/&(?:lt);/g, '<')
			.replace(/&(?:gt);/g, '>')
			.replace(/&(?:quot);/g, '"')
			.replace(/&(?:#39);/g, "'")
			// remove citations
			.replace(/\[\d+\]/ig, '');
	}

const singleTrailings = ['을', '를', '은', '는', '이', '가', '의', '에', '라', '와', '과', '도', '만', '한', '.', ','];
const doubleTrailings = ['에는', '에서', '에게', '이며', '까지', '다.'];

const getTokens = (text:string) => {
	const tokens = [];
	text.split(/\s/g).forEach(word => {
		if (word.trim() === '') {
			return;
		}
		let trailing = '';
		if (word.length > 2) {
			const lastTwoLetters = word.substring(word.length -2, word.length)
			if (doubleTrailings.indexOf(lastTwoLetters) !== -1) {
				word = word.substring(0, word.length - 2);
				trailing = lastTwoLetters;
			}
		}

		if (trailing === '') {
			const lastLetter = word[word.length - 1];
			if (singleTrailings.indexOf(lastLetter) !== -1) {
				word = word.substring(0, word.length - 1);
				trailing = lastLetter;
			}
		}

		wholeText += ` ${word}`;

		tokens.push({
			value: word,
			trailing: trailing,
			redacted: !solved && !guesses.hasOwnProperty(word),
		});
	});
	return tokens;
}

	const getParagraphsWithTokens = (text: string) => {
		const paragraphs = [];
		text.split(/\n/g).forEach(paragraph => {
			if (paragraph.trim() === '') {
				return;
			}
			const tokens = getTokens(paragraph);
			paragraphs.push(tokens);
		});

		return paragraphs;
	}

	getArticle();

	const handleGuessSubmit = () => {
		guess = guess.trim();
		if (guesses[guess]) {
			guess = '';
			return;
		}
		let regex = new RegExp(`(?:^|\\s|\\"|\\'|\\()${guess}(?:$|\\s|\\.|\\!|\\?|\\:|\\,|\\;|\\"|\\'|\\))`,'g');
		let count = (wholeText.match(regex) || []).length;
		guesses[guess] = count;
		guess = '';
		renderArticle();
	}
</script>

<svelte:head>
	<title>코리댁틀</title>
	<meta name="Koredactle" content="Koredactle" />
</svelte:head>

<header>
	<h1>
		Koredactle
	</h1>
</header>
<main>
	<article class={!wikiData ? 'loading' : undefined}>
			{#if !wikiData}
				<span>Loading....</span>
			{/if}
			{#each sections as section}
			<h3>
				{#each section.heading as headingToken}
					<Span value={headingToken.value} redacted={headingToken.redacted} trailing={headingToken.trailing}></Span>
					<span> </span>
				{/each}
			</h3>
				{#each section.paragraphs as paragraph}
					<p>
					{#each paragraph as token}
						<Span value={token.value} redacted={token.redacted} trailing={token.trailing}></Span>
						<span> </span>
					{/each}
					</p>
				{/each}
			{/each}
	</article>
	<section class="guesses">
		<form on:submit|preventDefault={handleGuessSubmit}>
			<label for="guess">예상 단어</label>
			<input id="guess" bind:value={guess}>
			<button type="submit">입력</button>
			{#if Object.keys(guesses).length > 0 }
				<span class="guess-count">총 개수: {Object.keys(guesses).length}</span>
			{/if}
		</form>
		<table class="guesses">
			{#each Object.keys(guesses).reverse() as guess}
				<tr>
					<td>{guess}</td>
					<td>{guesses[guess]}</td>
				</tr>
			{/each}
		</table>
	</section>
</main>
<style>
	main {
		display: flex;
		flex-direction: row;
    align-items: start;
		padding: 1rem;
		width: 100%;
		max-width: 1024px;
		margin: 0 auto;
		box-sizing: border-box;
		gap: 8px;
	}


	article.loading {
    width: 100%;
    margin: auto;
    text-align: center;
	}

	h1 {
		width: 100%;
    text-align: center;
	}

	section.guesses {
		position: fixed;
    background: #212529;
    bottom: 0;
    left: 0;
    padding: 0 24px 12px 24px;
    width: 100%;
    height: 30vh;
    overflow-y: scroll;
	}

	form {
		position: fixed;
		background: #212529;
    padding: 12px 0;
    width: 100%;
	}

	label {
		font-size: 14px;
	}

	input {
    font-size: 16px;
	}
	.guess-count {
		position: fixed;
    right: 12px;
		font-size: 14px;
	}
	table.guesses:not(:empty) {
		width: 100%;
		margin-top: 48px;
	}

@media (min-width: 768px) {
	article {
		padding-right: 30vw;
	}

	form {
		box-sizing: border-box;
		height: 72px;
	}
	label {
		display: block;
		margin-bottom: 4px;
	}
	.guess-count {
		position: fixed;
		top: 12px;
	}
	section.guesses {
    box-sizing: border-box;
		width: 30vw;
		height: 100%;
		left: unset;
		right: 0;
		top: 0;
	}
	table.guesses:not(:empty) {
		margin-top: 72px;
	}
}

</style>
