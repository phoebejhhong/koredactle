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

const getTokens = (text:string) => {
	const tokens = [];
	text.split(/\s/g).forEach(word => {
		let trailing = '';
		const lastLetter = word[word.length - 1];
		if (['을', '를', '은', '는', '이', '가', '의', '에', '라'].indexOf(lastLetter) !== -1) {
			word = word.substring(0, word.length -1);
			trailing = lastLetter;
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
			const tokens = getTokens(paragraph);
			paragraphs.push(tokens);
		});

		return paragraphs;
	}

	getArticle();

	const handleGuessSubmit = () => {
		let regex = new RegExp(`(?:^|\\s|\\"|\\'|\\()${guess}(?:$|\\s|\\.|\\!|\\?|\\:|\\,|\\;|\\"|\\'|\\))`,'g');
		let count = (wholeText.match(regex) || []).length;
		guesses[guess] = count;
		guess = '';
		renderArticle();
	}
</script>

<svelte:head>
	<title>Home</title>
	<meta name="Koredactle" content="Koredactle" />
</svelte:head>

<header>
	<h1>
		Koredactle
	</h1>
</header>
<main>
	<article>
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
		</form>
		<table class="guesses">
			{#each Object.keys(guesses) as guess}
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

	section {
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
		flex: 1;
	}

	h1 {
		width: 100%;
    text-align: center;
	}

	section.guesses {
    display: flex;
    gap: 24px;
	}

	input {
    font-size: 16px;
	}
	table.guesses {
		width: 100%;
	}
</style>
