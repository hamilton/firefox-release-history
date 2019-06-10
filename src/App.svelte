<script>
import { onMount } from 'svelte';
import { fly, fade } from 'svelte/transition';
import { timeDay } from 'd3-time';
import { interpolate } from 'd3-interpolate';
import { timeFormat } from 'd3-time-format'

import Spinner from './Spinner.svelte'

const url = 'https://product-details.mozilla.org/1.0/all.json'

const releaseEvents = [
	{
		str: '2012-08-28',
		date: new Date('2012-08-28'),
		event: 'Silent Updates Introduced'
	}
]

const allHands = [
	{
		date: new Date('2010-07-05'),
		location: 'Whistler, BC, CA'
	},
	{
		date: new Date('2011-04-04'),
		location: 'Mountain View'
	},
	{
		date: new Date('2013-10-04'),
		label: 'Summit 2013',
		location: 'Brussels / Santa Clara / Toronto'
	},
	{
		date: new Date('2014-12-01'),
		location: 'Portland, Oregon, USA',
	},
	{
		date: new Date('2015-06-22'),
		location: 'Whistler, BC, CA',
	},
	{
		date: new Date('2015-12-07'),
		location: 'Orlando, FL, USA',
	},
	{
		date: new Date('2016-06-13'),
		location: 'London, UK',
	},
		{
		date: new Date('2016-12-05'),
		location: 'Hawaii, USA',
	},
	{
		date: new Date('2017-06-26'),
		location: 'San Francisco, CA, USA',
	},
	{
		date: new Date('2017-12-11'),
		location: 'Austin, TX, USA',
	},
	{
		date: new Date('2018-06-11'), 
		location: 'San Francisco, CA, USA',
	},
	{
		date: new Date('2018-12-03'), 
		location: 'Orlando, FL, USA',
	},
	{
		date: new Date('2019-06-17'),
		location: 'Whistler, BC, CA',
	},
]

let releases = [];
let releaseMap = {}
const releaseRequest = fetch(url)
	.then(r=>r.json())
	.then(data => data.releases)
	.then(data => {
		Object.keys(data).forEach(release => {
			data[release].str = data[release].date
			data[release].date = new Date(data[release].date)
		})
		return data
	})
	.then(data => {releases = data; return data})
	.then(data => {
		Object.keys(data).forEach(release => {
			const releaseInfo = data[release];
			const date = releaseInfo.str;
			if (!(date in releaseMap)) releaseMap[date] = []
			releaseMap[date].push(releaseInfo)
		})
	})

$: releaseSet = releases;

const range = (s, e) => [...Array(e-s).keys()].map(si=>s+si)

const fm = timeFormat('%Y-%m-%d')
const fmRollover = timeFormat("%B %d, %Y")

const getYear = (y) => timeDay.range(
		new Date(`${y}-01-01`),
		new Date(`${y+1}-01-01`)
	).map(date => ({date, str: fm(date)}))

const getRelease = (day) => releaseMap[day.str]

const getLastMajorRelease = day => {
	const dates = Object.keys(releaseMap).sort().filter(dtstr => {
		return releaseMap[dtstr].some(release => {
			return (dtstr <= day.str && release.category === 'major' && release.product === 'firefox')
		})
	})
	if (releaseMap[dates[dates.length-1]]) {
		return releaseMap[dates[dates.length-1]][0]
	}
	return undefined;
	
}

const getAllHands = (day) => {
	return allHands.find(ah => {
		const start = ah.date;
		const end = new Date(+ah.date)
		end.setDate(end.getDate()+5)
		return day.date >= start && day.date <= end
	})
}

let yearSet = []

$: if(Object.keys(releaseMap).length) yearSet = range(2004, 2020)
	.map(getYear)
	.map(year => {
		return year.map(day => {
		// this is where we annotate the day.
		const lastMajorRelease = getLastMajorRelease(day)
		day.lastRelease = lastMajorRelease
		// was there a release?
		const releases = getRelease(day)
		if (releases) {
			day.releases = releases
			day.majorFFRelease = releases.some(release=>release.category==='major')
			day.onlyMinorReleases = !day.majorFFRelease
		}
		// was there an all-hands?
		const allHandsDate = getAllHands(day)
		if (allHandsDate) {
			day.allHands = allHandsDate
		}
		// add last major release here.
		return day
		})
	}
	)
// interate through yearSet and annotate with (1) releases

// DEPRECATED

const isReleaseDate = (date) => {
	return Object.keys(releaseSet)
	.some((k) => {
		return releaseSet[k].str === date.str
	})
}

let thisDay;
let thisMajorRelease;

function isInFocusedRelease (dayLastReleaseStr, compare) {
	return compare && compare === dayLastReleaseStr
	//return day.lastRelease && compare && day.lastRelease && compare === day.lastRelease.str
}

function show(str, lastReleaseStr) {
	if (lastReleaseStr) {
		yearSet.forEach(year => {
			year.forEach(day => {
				if (day.str === str) thisDay = day;
			})
		})
		thisMajorRelease = lastReleaseStr;
		document.querySelectorAll(`.release-${thisMajorRelease}`).forEach(dayNode => {
			dayNode.classList.add('in-focus')
		})
	} else {
		document.querySelectorAll(`.release-${thisMajorRelease}`).forEach(dayNode => {
			dayNode.classList.remove('in-focus')
		})
		thisMajorRelease = undefined;
		thisDay = undefined;
	}
}

// when mounted, then reveal.
let visible = false;
onMount(() => {visible = true;})

</script>

<style>

:root {
	--w: 8px;
}

main {
	width: max-content;
	margin: auto;
}

h1 {
	font-weight:900;
	text-transform: uppercase;
}

h1 span {
	font-weight: 100;
}

div.give-em-years {
	margin-top:calc(var(--w) * 5);
	margin-bottom:calc(var(--w) * 10);
}

div.year-set {
	display: grid;
	grid-template-columns: max-content max-content;
	grid-gap: var(--w);
	grid-column-gap: calc(var(--w));
	margin: auto;
	width: max-content;
}

div.year {
	display: grid;
	grid-template-columns: max-content max-content;
}

div.year-label {
	writing-mode: vertical-rl;
	text-orientation: mixed;
	text-align: center;
	font-size:12px;
	padding-right: var(--w);
	font-weight:900;
	opacity: .4;
}

div.year-container {
	display: grid;
	grid-template-columns: repeat(53, var(--w));
	grid-template-rows: repeat(7, var(--w));
	grid-auto-flow: column;
	grid-gap:1px;
}

div.day {
	background-color: 	#E8E8E8;
	width: var(--w);
	height: var(--w);
	border: 1px solid rgba(0,0,0,.1);
}

div.day:hover {
	transform: scale(1.5);
	border: 1px solid rgba(0,0,0,2);
	/* box-shadow: 0px 0px 10px rgba(0,0,0,1); */
}

div.rollover {
	min-height:100px;
}

.rollover-versions {
	display: flex;
}

.rollover-version {
	--rollover-fontsize: 14px;
	margin-right: var(--w);
	font-size: var(--rollover-fontsize);
}

.focused-release {
	border: 20px solid blue;
}

div.day.in-focus {
	background-color: rgb(146, 197, 238);
}

div.day.in-focus.minor-release {
	background-color: 	#8C92AC;
}

div.day.major-release {
	background-color: black;
}

div.day.minor-release {
	background-color: #B2BEB5;
}

div.day.all-hands {
	background-color: #FF7F7F;
}

div.day.all-hands.major-release {
	background-color:#C41E3A;
}

div.day.all-hands.minor-release {
	background-color: red;
}

</style>

<main>
{#await releaseRequest}
	<Spinner />
{:then _}
	{#if visible}
		<h1>
			The history of Firefox <span>WIP</span>
		</h1>
		<div class=rollover>
			{#if thisDay}
				{fmRollover(thisDay.date)}
				{#if thisDay.allHands}
					All-Hands {thisDay.allHands.location}
				{/if}
				{#if thisDay.releases}
					<div class=rollover-versions>
						{#each thisDay.releases as release}
							<div class=rollover-version>
								{release.product} {release.version}
							</div>
						{/each}
					</div>
				{/if}
			{/if}
		</div>
		<div in:fly={{y:20, duration: 400}} class=give-em-years>
			<div class=year-set>
				{#each yearSet as year, yi}
					<div
					class=year>
						{#if yi % 2 === 0}
							<div class=year-label>
								{yi + 2004}
							</div>
						{/if}
						<div class=year-container>
						{#each year as {allHands, lastRelease, releases, str, date, inFocus, majorFFRelease, onlyMinorReleases}, i (str)}
		
								<div
									style="
										grid-row:{i === 0 ? date.getDay()+1 : 'auto'};
									"
									class='day release-{lastRelease ? lastRelease.str : 'NONE'}'
									class:major-release={majorFFRelease}
									class:minor-release={onlyMinorReleases}
									class:all-hands={allHands}
									on:mouseover={(evt) => show(str, lastRelease.str)}
									on:mouseout={() => show()}
								/>
						{/each}
						</div>
					</div>
				{/each}
			</div>
		</div>
	{/if}
{/await}
</main>