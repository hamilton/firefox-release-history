<script>
import { onMount } from 'svelte';
import { fly, fade } from 'svelte/transition';
import { timeDay } from 'd3-time';
import { interpolate } from 'd3-interpolate';
import { timeFormat } from 'd3-time-format'

import Spinner from './Spinner.svelte'



//export let url = '/firefox-history.json'//'https://product-details.mozilla.org/1.0/all.json'
export let url = 'https://product-details.mozilla.org/1.0/all.json'


const TODAY = new Date();

const releaseEvents = [
	{
		str: '2012-08-28',
		date: new Date('2012-08-28'),
		event: 'Silent Updates Introduced'
	}
]

const daysBetween = (firstDate, secondDate) => {
    const oneDay = 24*60*60*1000; // hours*minutes*seconds*milliseconds
    return Math.round(Math.abs((firstDate.getTime() - secondDate.getTime())/(oneDay)));
}

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

// releaseRequest is the promise that contains all the
// version data.
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

// this is probably not necessary! I'm hacking though.
$: releaseSet = releases;
$: firstDate = Object.keys(releaseMap).sort()[0]

// yearning for a range function in js.
const range = (s, e) => [...Array(e-s).keys()].map(si=>s+si)

// formatters for date objects, here and there.
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
		return releaseMap[dates[dates.length-1]].find(release => release.category === 'major' && release.product === 'firefox')
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

// this is the major reactive piece.
// Once releaseMap gets some additional assignment,
// it regenerates
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
			day.majorFFRelease = releases.some(release=>release.category==='major' && release.product==='firefox')
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
}

function show(str, lastReleaseStr) {
	if (str) {
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

<main>
{#await releaseRequest}
	<Spinner />
{:then _}
	{#if visible}
		<h1 in:fly={{y:-10, duration:600}}>
			A History of Firefox Versions <span>WIP</span>
		</h1>
		<div in:fly={{y:-10, delay: 200, duration: 400}} class=rollover>
			{#if thisDay}
				<div class=rollover-date>
					{fmRollover(thisDay.date)}
								{#if thisDay.lastRelease}
					{#if daysBetween(thisDay.date, thisDay.lastRelease.date)}
						<span>
							{daysBetween(thisDay.date, thisDay.lastRelease.date)} days since {thisDay.lastRelease.product} {thisDay.lastRelease.version}
						</span>
					{/if}
				{/if}
				</div>
	
				{#if thisDay.allHands}
					<div class=rollover-allhands>
						All-Hands {thisDay.allHands.location}
					</div>
				{/if}
				{#if thisDay.releases}
					<div class=rollover-versions>
						{#each thisDay.releases as release}
							<div class=rollover-version class:ff-major-rollover={release.category === 'major' && release.product === 'firefox'}>
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
									class='day release-{(lastRelease && date < TODAY) ? lastRelease.str : 'NONE'}'
									class:major-release={majorFFRelease}
									class:minor-release={onlyMinorReleases}
									class:inactive-date={str < firstDate || date > TODAY}
									class:all-hands={allHands}
									on:mouseover={(evt) => show(str, lastRelease ? lastRelease.str : undefined)}
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