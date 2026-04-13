<script lang="ts">
	import * as Card from '$lib/components/ui/card';
	import { Label } from '$lib/components/ui/label';
	import { Input } from '$lib/components/ui/input';

	type NedbetalingRad = {
		betalingNr: number;
		ar: number;
		maned: number;
		terminbelop: number;
		avdrag: number;
		renter: number;
		restgjeld: number;
	};

	const currencyFormatter = new Intl.NumberFormat('nb-NO', {
		style: 'currency',
		currency: 'NOK',
		maximumFractionDigits: 0
	});
	const aksjeSkattVedUttak = 0.3784;

	function roundToTwoDecimals(value: number) {
		return Math.round((value + Number.EPSILON) * 100) / 100;
	}

	function formatCurrency(value: number) {
		return currencyFormatter.format(value);
	}

	function beregnSluttverdiForEngangsbelop(
		innskuttBelop: number,
		arligAvkastningProsent: number,
		antallAr: number
	) {
		return innskuttBelop * Math.pow(1 + arligAvkastningProsent / 100, antallAr);
	}

	function beregnSluttverdiForManedligSparing(
		innskuddPerManed: number,
		arligAvkastningProsent: number,
		antallManeder: number
	) {
		if (innskuddPerManed <= 0 || antallManeder === 0) {
			return 0;
		}

		const manedligAvkastning = Math.pow(1 + arligAvkastningProsent / 100, 1 / 12) - 1;

		if (manedligAvkastning === 0) {
			return innskuddPerManed * antallManeder;
		}

		return (
			innskuddPerManed *
			((Math.pow(1 + manedligAvkastning, antallManeder) - 1) / manedligAvkastning)
		);
	}

	function beregnEtterSkattSluttverdi(sluttverdiForSkatt: number, innskuttBelop: number) {
		const gevinst = sluttverdiForSkatt - innskuttBelop;

		if (gevinst <= 0) {
			return sluttverdiForSkatt;
		}

		return sluttverdiForSkatt - gevinst * aksjeSkattVedUttak;
	}

	function byggNedbetalingstabell(
		lanebelop: number,
		arligRenteProsent: number,
		lopetidAr: number
	): NedbetalingRad[] {
		const hovedstol = Math.max(0, lanebelop);
		const antallBetalinger = Math.max(0, Math.round(lopetidAr * 12));

		if (hovedstol === 0 || antallBetalinger === 0) {
			return [];
		}

		const manedsrente = Math.max(0, arligRenteProsent) / 100 / 12;
		const terminbelop =
			manedsrente === 0
				? hovedstol / antallBetalinger
				: (hovedstol * manedsrente) / (1 - Math.pow(1 + manedsrente, -antallBetalinger));

		const tabell: NedbetalingRad[] = [];
		let restgjeld = hovedstol;

		for (let betalingNr = 1; betalingNr <= antallBetalinger; betalingNr += 1) {
			const renter = manedsrente === 0 ? 0 : restgjeld * manedsrente;
			let avdrag = terminbelop - renter;
			let faktiskTerminbelop = terminbelop;

			if (betalingNr === antallBetalinger) {
				avdrag = restgjeld;
				faktiskTerminbelop = avdrag + renter;
			}

			restgjeld = Math.max(0, restgjeld - avdrag);

			tabell.push({
				betalingNr,
				ar: Math.ceil(betalingNr / 12),
				maned: ((betalingNr - 1) % 12) + 1,
				terminbelop: roundToTwoDecimals(faktiskTerminbelop),
				avdrag: roundToTwoDecimals(avdrag),
				renter: roundToTwoDecimals(renter),
				restgjeld: roundToTwoDecimals(restgjeld)
			});
		}

		return tabell;
	}

	// Shared
	let tidshorisont = $state(2);

	// Renting
	let manedsleie = $state(22000);
	let forventetAvkastning = $state(10);

	// Buying
	let kjopesum = $state(5000000);
	let egenkapital = $state(500000);
	let rente = $state(4.8);
	let nedbetalingstid = $state(30);
	let boligprisvekst = $state(6);
	let felleskostnadPerManed = $state(2000);
	let kommunaleAvgifterPerAr = $state(12000);
	let omkostningerKjop = $state(2.5);
	let omkostningerSalg = $state(2.5);

	let lanebelop = $derived(Math.max(0, kjopesum - egenkapital));
	let nedbetalingstabell = $derived.by(() =>
		byggNedbetalingstabell(lanebelop, rente, nedbetalingstid)
	);
	let manedligTerminbelop = $derived(nedbetalingstabell[0]?.terminbelop ?? 0);
	let tidshorisontIMnd = $derived(
		Math.max(0, Math.min(nedbetalingstabell.length, Math.round(tidshorisont * 12)))
	);
	let betalingerInnenforTidshorisont = $derived(nedbetalingstabell.slice(0, tidshorisontIMnd));
	let totaleRenterInnenforTidshorisont = $derived.by(() =>
		betalingerInnenforTidshorisont.reduce((sum, rad) => sum + rad.renter, 0)
	);
	let renterEtterSkattefradrag = $derived(totaleRenterInnenforTidshorisont * 0.78);
	let boligverdiVedSalg = $derived(kjopesum * Math.pow(1 + boligprisvekst / 100, tidshorisont));
	let gevinstAvVerdistigning = $derived(boligverdiVedSalg - kjopesum);
	let totaleFelleskostnader = $derived(felleskostnadPerManed * tidshorisontIMnd);
	let totaleKommunaleAvgifter = $derived(kommunaleAvgifterPerAr * tidshorisont);
	let omkostnadVedKjop = $derived(kjopesum * (omkostningerKjop / 100));
	let omkostnadVedSalg = $derived(boligverdiVedSalg * (omkostningerSalg / 100));
	let totalkostnadVedEie = $derived(
		renterEtterSkattefradrag +
			totaleFelleskostnader +
			totaleKommunaleAvgifter +
			omkostnadVedKjop +
			omkostnadVedSalg -
			gevinstAvVerdistigning
	);
	let totalLeieIPerioden = $derived(manedsleie * tidshorisontIMnd);
	let sluttverdiEgenkapitalinvestering = $derived(
		beregnSluttverdiForEngangsbelop(egenkapital, forventetAvkastning, tidshorisont)
	);
	let gevinstEtterSkattPaEgenkapital = $derived(
		beregnEtterSkattSluttverdi(sluttverdiEgenkapitalinvestering, egenkapital) - egenkapital
	);
	let manedligSparingVedLeie = $derived(Math.max(0, manedligTerminbelop - manedsleie));
	let innskuttManedligSparing = $derived(manedligSparingVedLeie * tidshorisontIMnd);
	let sluttverdiAvManedligSparing = $derived(
		beregnSluttverdiForManedligSparing(
			manedligSparingVedLeie,
			forventetAvkastning,
			tidshorisontIMnd
		)
	);
	let etterSkattVerdiAvManedligSparing = $derived(
		beregnEtterSkattSluttverdi(sluttverdiAvManedligSparing, innskuttManedligSparing)
	);
	let totalkostnadVedLeie = $derived(
		totalLeieIPerioden - gevinstEtterSkattPaEgenkapital - etterSkattVerdiAvManedligSparing
	);
</script>

<div class="min-h-screen bg-background">
	<div class="mx-auto max-w-5xl px-6 py-12">
		<header class="mb-10">
			<h1 class="text-3xl font-bold tracking-tight text-foreground">Boligkalkulator</h1>
			<p class="mt-2 text-sm text-muted-foreground">
				Sammenlign kostnadene ved å leie vs. kjøpe bolig over tid.
			</p>
		</header>

		<!-- Shared: Tidshorisont -->
		<div class="mb-8">
			<Card.Root>
				<Card.Header>
					<Card.Title>Tidshorisont</Card.Title>
					<Card.Description>Hvor lenge planlegger du å bo der?</Card.Description>
				</Card.Header>
				<Card.Content>
					<div class="flex items-center gap-3">
						<div class="w-48">
							<Label for="tidshorisont" class="mb-1.5 block text-sm">Antall år</Label>
							<div class="relative">
								<Input
									id="tidshorisont"
									type="number"
									min="1"
									max="50"
									placeholder="10"
									class="pr-12"
									bind:value={tidshorisont}
								/>
								<span
									class="pointer-events-none absolute inset-y-0 right-3 flex items-center text-sm text-muted-foreground"
									>år</span
								>
							</div>
						</div>
					</div>
				</Card.Content>
			</Card.Root>
		</div>

		<!-- Two-column layout -->
		<div class="grid grid-cols-1 gap-6 lg:grid-cols-2">
			<!-- Leie -->
			<Card.Root>
				<Card.Header>
					<div class="flex items-center gap-2">
						<span
							class="inline-flex items-center rounded bg-muted px-2 py-0.5 text-xs font-semibold tracking-widest text-muted-foreground uppercase"
						>
							Leie
						</span>
					</div>
					<Card.Title class="mt-1">Leiealternativet</Card.Title>
					<Card.Description>Hva koster det å leie, og hva gjør du med resten?</Card.Description>
				</Card.Header>
				<Card.Content class="space-y-5">
					<div>
						<Label for="manedsleie" class="mb-1.5 block text-sm">Månedsleie</Label>
						<div class="relative">
							<Input
								id="manedsleie"
								type="number"
								min="0"
								placeholder="22000"
								class="pr-14"
								bind:value={manedsleie}
							/>
							<span
								class="pointer-events-none absolute inset-y-0 right-3 flex items-center text-sm text-muted-foreground"
								>kr/mnd</span
							>
						</div>
					</div>

					<div>
						<Label for="avkastning" class="mb-1.5 block text-sm"
							>Forventet avkastning aksjemarked</Label
						>
						<div class="relative">
							<Input
								id="avkastning"
								type="number"
								min="0"
								max="100"
								step="0.1"
								placeholder="10"
								class="pr-10"
								bind:value={forventetAvkastning}
							/>
							<span
								class="pointer-events-none absolute inset-y-0 right-3 flex items-center text-sm text-muted-foreground"
								>%</span
							>
						</div>
						<p class="mt-1.5 text-xs text-muted-foreground">Per år</p>
					</div>
				</Card.Content>
			</Card.Root>

			<!-- Kjøp -->
			<Card.Root>
				<Card.Header>
					<div class="flex items-center gap-2">
						<span
							class="inline-flex items-center rounded bg-foreground px-2 py-0.5 text-xs font-semibold tracking-widest text-background uppercase"
						>
							Kjøp
						</span>
					</div>
					<Card.Title class="mt-1">Kjøpsalternativet</Card.Title>
					<Card.Description>Hva koster det å eie boligen?</Card.Description>
				</Card.Header>
				<Card.Content class="space-y-5">
					<div>
						<Label for="kjopesum" class="mb-1.5 block text-sm">Kjøpesum</Label>
						<div class="relative">
							<Input
								id="kjopesum"
								type="number"
								min="0"
								placeholder="5000000"
								class="pr-6"
								bind:value={kjopesum}
							/>
							<span
								class="pointer-events-none absolute inset-y-0 right-3 flex items-center text-sm text-muted-foreground"
								>kr</span
							>
						</div>
					</div>

					<div>
						<Label for="egenkapital" class="mb-1.5 block text-sm">Egenkapital</Label>
						<div class="relative">
							<Input
								id="egenkapital"
								type="number"
								min="0"
								placeholder="1000000"
								class="pr-6"
								bind:value={egenkapital}
							/>
							<span
								class="pointer-events-none absolute inset-y-0 right-3 flex items-center text-sm text-muted-foreground"
								>kr</span
							>
						</div>
					</div>

					<div>
						<Label for="rente" class="mb-1.5 block text-sm">Rente på boliglån</Label>
						<div class="relative">
							<Input
								id="rente"
								type="number"
								min="0"
								max="100"
								step="0.01"
								placeholder="4.8"
								class="pr-10"
								bind:value={rente}
							/>
							<span
								class="pointer-events-none absolute inset-y-0 right-3 flex items-center text-sm text-muted-foreground"
								>%</span
							>
						</div>
						<p class="mt-1.5 text-xs text-muted-foreground">Nominell rente per år</p>
					</div>

					<div>
						<Label for="nedbetalingstid" class="mb-1.5 block text-sm"
							>Nedbetalingstid for boliglån</Label
						>
						<div class="relative">
							<Input
								id="nedbetalingstid"
								type="number"
								min="1"
								max="50"
								placeholder="30"
								class="pr-12"
								bind:value={nedbetalingstid}
							/>
							<span
								class="pointer-events-none absolute inset-y-0 right-3 flex items-center text-sm text-muted-foreground"
								>år</span
							>
						</div>
					</div>

					<div>
						<Label for="boligprisvekst" class="mb-1.5 block text-sm">Forventet boligprisvekst</Label
						>
						<div class="relative">
							<Input
								id="boligprisvekst"
								type="number"
								min="0"
								max="100"
								step="0.1"
								placeholder="6"
								class="pr-10"
								bind:value={boligprisvekst}
							/>
							<span
								class="pointer-events-none absolute inset-y-0 right-3 flex items-center text-sm text-muted-foreground"
								>%</span
							>
						</div>
						<p class="mt-1.5 text-xs text-muted-foreground">Per år</p>
					</div>

					<div>
						<Label for="felleskostnad" class="mb-1.5 block text-sm">Felleskostnad per mnd</Label>
						<div class="relative">
							<Input
								id="felleskostnad"
								type="number"
								min="0"
								placeholder="2000"
								class="pr-14"
								bind:value={felleskostnadPerManed}
							/>
							<span
								class="pointer-events-none absolute inset-y-0 right-3 flex items-center text-sm text-muted-foreground"
								>kr/mnd</span
							>
						</div>
					</div>

					<div>
						<Label for="kommunale-avgifter" class="mb-1.5 block text-sm"
							>Kommunale avgifter per år</Label
						>
						<div class="relative">
							<Input
								id="kommunale-avgifter"
								type="number"
								min="0"
								placeholder="12000"
								class="pr-10"
								bind:value={kommunaleAvgifterPerAr}
							/>
							<span
								class="pointer-events-none absolute inset-y-0 right-3 flex items-center text-sm text-muted-foreground"
								>kr</span
							>
						</div>
						<p class="mt-1.5 text-xs text-muted-foreground">Per år</p>
					</div>

					<div class="grid grid-cols-2 gap-4">
						<div>
							<Label for="omkostninger-kjop" class="mb-1.5 block text-sm">Omkostninger kjøp</Label>
							<div class="relative">
								<Input
									id="omkostninger-kjop"
									type="number"
									min="0"
									max="100"
									step="0.1"
									placeholder="2.5"
									class="pr-10"
									bind:value={omkostningerKjop}
								/>
								<span
									class="pointer-events-none absolute inset-y-0 right-3 flex items-center text-sm text-muted-foreground"
									>%</span
								>
							</div>
						</div>

						<div>
							<Label for="omkostninger-salg" class="mb-1.5 block text-sm">Omkostninger salg</Label>
							<div class="relative">
								<Input
									id="omkostninger-salg"
									type="number"
									min="0"
									max="100"
									step="0.1"
									placeholder="2.5"
									class="pr-10"
									bind:value={omkostningerSalg}
								/>
								<span
									class="pointer-events-none absolute inset-y-0 right-3 flex items-center text-sm text-muted-foreground"
									>%</span
								>
							</div>
						</div>
					</div>
				</Card.Content>
			</Card.Root>
		</div>

		<div class="mt-8 grid grid-cols-1 gap-6 xl:grid-cols-2">
			<Card.Root>
				<Card.Header>
					<Card.Title>Kostnader ved leie</Card.Title>
					<Card.Description>
						Leieutgifter minus investering av egenkapital og månedlig differanse mot terminbeløpet.
					</Card.Description>
				</Card.Header>
				<Card.Content class="space-y-4 text-sm">
					<div class="grid gap-3 sm:grid-cols-2">
						<div class="rounded-lg border p-3">
							<p class="text-xs text-muted-foreground">Leie i perioden</p>
							<p class="mt-1 font-medium text-foreground">{formatCurrency(totalLeieIPerioden)}</p>
						</div>

						<div class="rounded-lg border p-3">
							<p class="text-xs text-muted-foreground">Avkastning på investert egenkapital</p>
							<p class="mt-1 font-medium text-foreground">
								{formatCurrency(gevinstEtterSkattPaEgenkapital)}
							</p>
						</div>

						<div class="rounded-lg border p-3">
							<p class="text-xs text-muted-foreground">Månedlig spart beløp</p>
							<p class="mt-1 font-medium text-foreground">
								{formatCurrency(manedligSparingVedLeie)}
							</p>
						</div>

						<div class="rounded-lg border p-3">
							<p class="text-xs text-muted-foreground">Oppspart verdi av månedsdifferanse</p>
							<p class="mt-1 font-medium text-foreground">
								{formatCurrency(etterSkattVerdiAvManedligSparing)}
							</p>
						</div>

						<div class="rounded-lg border bg-muted/40 p-3 sm:col-span-2">
							<p class="text-xs text-muted-foreground">Totalkostnad ved leie</p>
							<p class="mt-1 text-lg font-semibold text-foreground">
								{formatCurrency(totalkostnadVedLeie)}
							</p>
						</div>
					</div>
				</Card.Content>
			</Card.Root>

			<Card.Root>
				<Card.Header>
					<Card.Title>Kostnader ved eie</Card.Title>
					<Card.Description>
						Annuitetslån med månedlige betalinger basert på kjøpesum minus egenkapital.
					</Card.Description>
				</Card.Header>
				<Card.Content class="space-y-4 text-sm">
					<div class="grid gap-3 sm:grid-cols-2">
						<div class="rounded-lg border p-3">
							<p class="text-xs text-muted-foreground">Månedlig terminbeløp</p>
							<p class="mt-1 font-medium text-foreground">{formatCurrency(manedligTerminbelop)}</p>
						</div>

						<div class="rounded-lg border p-3">
							<p class="text-xs text-muted-foreground">Renter etter skattefradrag</p>
							<p class="mt-1 font-medium text-foreground">
								{formatCurrency(renterEtterSkattefradrag)}
							</p>
						</div>

						<div class="rounded-lg border p-3">
							<p class="text-xs text-muted-foreground">Gevinst av verdistigning</p>
							<p class="mt-1 font-medium text-foreground">
								{formatCurrency(gevinstAvVerdistigning)}
							</p>
						</div>

						<div class="rounded-lg border p-3">
							<p class="text-xs text-muted-foreground">Felleskostnader i perioden</p>
							<p class="mt-1 font-medium text-foreground">
								{formatCurrency(totaleFelleskostnader)}
							</p>
						</div>

						<div class="rounded-lg border p-3">
							<p class="text-xs text-muted-foreground">Kommunale avgifter i perioden</p>
							<p class="mt-1 font-medium text-foreground">
								{formatCurrency(totaleKommunaleAvgifter)}
							</p>
						</div>

						<div class="rounded-lg border p-3">
							<p class="text-xs text-muted-foreground">Omkostninger ved kjøp</p>
							<p class="mt-1 font-medium text-foreground">
								{formatCurrency(omkostnadVedKjop)}
							</p>
						</div>

						<div class="rounded-lg border p-3">
							<p class="text-xs text-muted-foreground">Omkostninger ved salg</p>
							<p class="mt-1 font-medium text-foreground">
								{formatCurrency(omkostnadVedSalg)}
							</p>
						</div>

						<div class="rounded-lg border bg-muted/40 p-3 sm:col-span-2">
							<p class="text-xs text-muted-foreground">Totalkostnad ved eie</p>
							<p class="mt-1 text-lg font-semibold text-foreground">
								{formatCurrency(totalkostnadVedEie)}
							</p>
						</div>
					</div>
				</Card.Content>
			</Card.Root>
		</div>
	</div>
</div>
