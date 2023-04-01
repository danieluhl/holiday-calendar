<script>
	const DAYS = ['Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday'];
	const bindByDay =
		({ month, day }) =>
		(year, sat, sun) =>
			getHolidayDateByDay({ year, sat, sun, month: month - 1, day });
	const bindByNth =
		({ month, weekday, nth }) =>
		(year, sat, sun) =>
			getHolidayByNth({ year, sat, sun, month: month - 1, weekday, nth });

	const holidays = {
		'🎇 New Years': bindByDay({ month: 1, day: 1 }),
		'Martin Luther King Day': bindByNth({ month: 1, weekday: DAYS.indexOf('Monday'), nth: 3 }),
		"🇺🇸 Presidents' Day": bindByNth({ month: 2, weekday: DAYS.indexOf('Monday'), nth: 3 }),
		'🇺🇸 Memorial Day': bindByNth({ month: 5, weekday: DAYS.indexOf('Monday'), nth: -1 }),
		Juneteenth: bindByDay({ month: 6, day: 19 }),
		'🎇 Independence Day': bindByDay({ month: 7, day: 4 }),
		'Labor Day': bindByNth({ month: 9, weekday: DAYS.indexOf('Monday'), nth: 1 }),
		"Indigenous People's Day": bindByNth({ month: 10, weekday: DAYS.indexOf('Monday'), nth: 2 }),
		'🇺🇸 Veterens Day': bindByDay({ month: 11, day: 11 }),
		'🦃 Thanksgiving Day': bindByNth({ month: 11, weekday: DAYS.indexOf('Thursday'), nth: 4 }),
		'🦃 Day after Thanksgiving': bindByNth({ month: 11, weekday: DAYS.indexOf('Friday'), nth: 4 }),
		'☃️ Christmas Day': bindByDay({ month: 12, day: 25 })
	};

	let icsResults = '';
	let description = '';
	const getHolidayDateByDay = ({ year, sat, sun, month, day }) => {
		// get the date
		const d = new Date();
		d.setFullYear(year);
		d.setMonth(month);
		d.setDate(day);
		// check if sat or sun
		if (d.getDay() === 0) {
			d.setDate(d.getDate() + sun);
		}
		if (d.getDay() === 6) {
			d.setDate(d.getDate() + sat);
		}
		// return the full date
		return d;
	};
	const getHolidayByNth = ({ year, sat, sun, month, weekday, nth }) => {
		// get the date
		const d = new Date();
		d.setFullYear(year);
		d.setMonth(month);
		d.setDate(1);
		const dayOfWeek = d.getDay();
		// if nth = -1, do last of that day in the month
		if (nth === -1) {
			d.setMonth(month + 1); // go one month forward
			d.setDate(d.getDate() - 1);
			while (d.getDay() !== weekday) {
				d.setDate(d.getDate() - 1);
			}
		} else {
			const nthDay = ((weekday + 7 - dayOfWeek) % 7) + 7 * (nth - 1);
			d.setDate(nthDay + 1);
		}
		// check if sat or sun
		if (d.getDay() === 0) {
			d.setDate(d.getDate() + sun);
		}
		if (d.getDay() === 6) {
			d.setDate(d.getDate() + sat);
		}
		// return the full date
		return d;
	};

	const thisYear = new Date().getFullYear();
	const years = Array.from(new Array(10), (_, i) => thisYear + i);
	let selectedYear = thisYear;

	const handleSubmit = (e) => {
		const formData = new FormData(e.target);
		const data = {};
		const selectedHolidays = [];
		for (let field of formData) {
			const [key, value] = field;
			if (key.startsWith('holiday')) {
				selectedHolidays.push(key.replace('holiday-', ''));
			} else {
				data[key] = value;
			}
		}
		const holidayDates = selectedHolidays.map((holiday) => {
			const holidayDate = holidays[holiday](
				data.year,
				data.saturdayRules === 'friday' ? -1 : 2,
				data.sundayRules === 'friday' ? -2 : 1
			);

			return [holiday, holidayDate];
		});
		// for each date, generate the stuff
		icsResults = buildCalendar(holidayDates);
	};
	const buildCalendar = (holidayDates) => {
		const prefix = `BEGIN:VCALENDAR
VERSION:2.0
`;
		const postfix = `END:VCALENDAR`;
		const icsHolidays = holidayDates
			.map(([label, date]) => {
				return `BEGIN:VEVENT
DTSTART;VALUE=DATE:${date.getFullYear()}${(date.getMonth() + 1).toString().padStart(2, '0')}${date
					.getDate()
					.toString()
					.padStart(2, '0;')}
DTEND;VALUE=DATE:${date.getFullYear()}${(date.getMonth() + 1).toString().padStart(2, '0;')}${(
					date.getDate() + 1
				)
					.toString()
					.padStart(2, '0;')}
SUMMARY:${label}
SEQUENCE:0${description ? `\nDESCRIPTION:${description}` : ''}
END:VEVENT
`;
			})
			.join('\n');

		return `${prefix}${icsHolidays}${postfix}`;
	};

	function handleDownload(filename, text) {
		var element = document.createElement('a');
		element.setAttribute('href', 'data:text/plain;charset=utf-8,' + encodeURIComponent(text));
		element.setAttribute('download', filename);

		element.style.display = 'none';
		document.body.appendChild(element);

		element.click();

		document.body.removeChild(element);
	}

	// NICE TO HAVES
	// override rules by holiday option
	// mark federal holidays
	// add random holidays
</script>

<main class="py-10 m-auto max-w-xl text-gray-300 bg-slate-800">
	<h1>Holiday Calendar Generator</h1>
	<p class="pt-3">
		Business treat different holidays differnently. This tool will help you generate a `.ics` file
		with all the holidays your company does for the year landing on the right days you have off. You
		can share this file around with the rest of your company or get a link to the file here (coming
		soon!).
	</p>

	<form class="flex flex-col items-start pt-10" on:submit|preventDefault={handleSubmit}>
		<h2>Select Year</h2>
		<select
			class="py-2 px-5 my-3 rounded-md ring-2 ring-gray-600 bg-slate-900"
			id="year"
			name="year"
		>
			{#each years as year}
				<option value={year}>{year}</option>
			{/each}
		</select>

		<h2 class="pt-5">Select Holidays</h2>
		<subheading>Please select which holidays your company recognizes</subheading>
		<ul class="py-5">
			{#each Object.keys(holidays) as holiday}
				<li class="pb-1 text-lg">
					<label>
						<input
							class="mr-3 text-gray-300 bg-slate-900"
							name={`holiday-${holiday}`}
							data-type="holiday"
							type="checkbox"
							id={holiday}
							checked
						/>{holiday}
					</label>
				</li>
			{/each}
		</ul>
		<h2 class="pt-5">Select Rules</h2>
		<p class="py-3">If a holiday lands on Saturday, what day do you take off?</p>
		<select
			name="saturdayRules"
			class="py-2 px-5 my-3 rounded-md ring-2 ring-gray-600 bg-slate-900"
		>
			<option value="friday" selected>Friday</option>
			<option value="monday">Monday</option>
		</select>
		<p>What about for a Sunday?</p>
		<select name="sundayRules" class="py-2 px-5 my-3 rounded-md ring-2 ring-gray-600 bg-slate-900">
			<option value="friday">Friday</option>
			<option value="monday" selected>Monday</option>
		</select>
		<p>Optional description applied to all events (e.g. "company holiday event")</p>
		<input
			class="self-stretch py-2 px-5 my-3 max-w-xl rounded-md ring-2 ring-gray-600 bg-slate-900"
			type="text"
			value={description}
		/>
		<input
			class="self-end py-2 px-5 my-3 bg-green-800 rounded-md ring-2 ring-green-600"
			type="submit"
			value="Generate"
		/>
	</form>

	<section class="flex flex-col pt-10">
		<hr class="border-black" />
		<hr class="border-gray-600" />
		<textarea
			class="my-3 mt-5 mb-2 max-w-xl rounded-md ring-2 ring-gray-600 bg-slate-900"
			rows="10"
			cols="80">{icsResults}</textarea
		>
		<input
			class="self-end py-2 px-5 my-3 bg-red-900 rounded-md ring-2 ring-red-600"
			type="submit"
			value="Download"
			on:click={handleDownload}
		/>
		<p class="text-gray-500">
			WARNING: this will add all the holidays at once but you'll have to remove them one at a time
			if there's an issue.
		</p>
	</section>
</main>

<style lang="postcss">
</style>
