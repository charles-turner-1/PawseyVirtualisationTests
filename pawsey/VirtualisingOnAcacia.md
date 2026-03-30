# Virtualising & Distibuting Data through Acacia (Pawsey) & Nirin (NCI)

At AMOS 2026 (Hobart, February), we did a bunch of alpha user testing on a new tool we've been working on, [the ACCESS-NRI Interactive Data Catalogue](https://access-nri.github.io/interactive-data-catalogue/#/). 
This is a tool that lets people click through the catalogue of data that we maintain on Gadi. The key insight that powers this is that the metadata that we trawl, index, and and make available in order to discover what climate data products are available on Gadi is on the order of megabytes, not petabytes, which means that we can ship it straight to the browser. As a result, we can stream data indices for around 20PB of data and let users click through them, virtually in real time.

This all sounds great, but it has a few limitations:
-  We use intake-esm to index our datasets, which are on disk on Gadi, NCI's HPC System. This produces parquet files containing metadata about the datasets in question, but those datasets are still unreachable. As a result, we are unable to display any richer information. 

    The pangeo catalog gets around this limitation by using a properly cloud native approach. The datasets, not just the indices are made available through object storage, and this means that they can dump a *full xarray repr* straight into the page (eg. https://catalog.pangeo.io/browse/master/atmosphere/era5_hourly_reanalysis_single_levels_sa/). This is, unfortunately, a major limitation of our Hybrid HPC/Cloud system.

    Unfortunately, if you can't get your hands on the `repr`, which is just an interactive view of the dataset metadata, our hopes of getting anything more interesting are also zero.

- We have to keep our two catalogues manually synced. NCI compute nodes don't have any internet access, and ARE sessions are throttled pretty heavily for reasons I still don't fully understand. Whilst you can create an intake-esm datastore with local paths and dump it into object storage (the paths will then only work if you download that datstore to the correct machine), the download performance on ARE makes this a generally fairly poor experience for the user. If they want to access a catalog in object storage from a compute node, they're totally out of luck. (Well, maybe not totally... NCI have their own cloud, Nirin, which compute nodes might be able to talk to?)

- Users still need to get themselves on NCI in order to access the data. This might be straightforward, if they're in a university research group with other NCI users, or they know someone with a compute allocation who is happy to add them to their project. In theory, anyone with an Australian academic or government email address should be able to get themselves onto NCI to look at this data (as far as I'm aware). In practice, this is going to limit data access to people who already know how to access it - despite our best efforts.


When we were gathering feedback on the tool, a few people mentioned that it would be really handy to actually look at the data itself when exploring the data. This is a much bigger task - it's a lot easier to ship a few megabytes than several petabytes. Actually shipping the data, rather than just a metadata index, means figuring out a way to let users actually download the slice of the data they're interested in looking at. This means you have a couple of hard requirements:
1. You cannot build a system that forces a user to download an entire netCDF file in order to look at a subset of it. If that netCDF file is 25GB, there's just no way of doing it in real time, even in the highly unrealistic case that you somehow manged to saturate the 10GB/s bandwith that is in theory available over ethernet.
2. You need to be able to anonymously & instantaneously access data. I have no real evidence of this, but I've clicked away from enough sites that have tried to get me register to access a resource, or decided that I don't care enough to watch the whole way through an advert on youtube, that I firmly believe this. In the TikTok era, you cannot get someone to register, select a slice of data, click 'Submit request', and then wait 5 minutes for a server to open, subset, serialise, and send you that slice. People just don't care that much.
3. The data needs to be on an HPC system, somehow. This is Australia, and we don't do cloud. Maybe in 10 years...

Squaring all these requirements would get a bit tough, if not for some of the amazing infrastructure work done by Tom Nicholas (virtualizarr) and Martin Durant (kerchunk) in particular, as well as the rest of the guys at Earthmover and carbonplan. Without all their hard work, this never would have gotten off the ground.

The second slice of luck is that the Pawsey supercomputing centre, 20 minutes drive from my house in Perth, WA, just so happened to implement an S3 compatible Ceph object store, *Acacia*, instead of a more traditional `gdata` area. Very lucky! 

When I first had a crack at this, I wasn't super sure about how easy it was going to be to get what I was after out of Acacia. I've mostly heard people describe it as weird, and/or hard to use. Apparently, it can be a little contrived to spin up a Jupyter sever on Setonix (the HPC proper) and have it read files in Acacia. Fortunately, that's not my use case - I was specifically interested about whether we could use this to ship data to *users who weren't on Acacia*.


___ 

## Virtualisation

### Zarr 

If you want to send these large array datasets over the wire (eg., stream it to a browser viewer widget), the only way to do it is Zarr. There have been various other solutions in the past - things like Thredds - but fundamentally, the user experience of those things is less than pleasant. 

Zarr is basically a rethink of netCDF/HDF, designed to be optimised for cloud storage and access. The problem is it really does not play nicely with HPC systems, especially those with a lustre filesystem (like NCI's Gadi machine).
If you take a typical ocean model run, with maybe 200 years of data spread across maybe 800 netCDF files, and reserialise it as zarr, you typically wind up creating hundred of thousands, maybe millions of inodes. Ignoring recent developments, this is basically because each chunk in the zarr store (the equivalent of all our netCDF files) gets its own inode.

On a lustre filesystem, this is a big no-no, and as a result, Zarr hasn't really taken off in Australia (remember, we don't do cloud). At that same AMOS, someone referred to zarr as an 'emerging format' - despite its already (extremely) widespread adoption, and being on version 3. I guess Python is an emerging programming language in some places too? 

### Virtualisating netCDF datasets

Virtualisation is basically the way to square the circle here. 
Virtualizarr lets you take a bunch of netCDF's and create a 'Virtual Zarr store' from them: something that you can query and work with as if were zarr, with the backend infrastructure handling fetching all the file chunks for you.
The idea here was that it means you can pop a bunch of netCDF files in the cloud - which is typically a fairly bad idea - see [this blog post](https://earthmover.io/blog/fundamentals-what-is-cloud-optimized-scientific-data) by Tom Nicholas - and then virtualise them, making it as fast to access them as if they were Zarr.

To my knowledge, Martin Durant (who originally conceived of kerchunk to do this), was mostly interested in the problem of getting these netCDF type files out of cloud object storage efficient. However, virtualisation (perhaps inadvertantly - I don't know the history) also solves a bunch of other problems.

- Opening large, multifile datasets. On a lustre file system, opening a file that hasn't been touched in a while can be really slow - like, upwards of a couple of seconds slow. This means that if you want to open a dataset with eg. 500 files, a really nontrivial amount of time (from the users perspective) can be spent *just opening files*. A kerchunk reference can be as simple as a single json file - so the you spend less time fighting the filesystem to get a `repr`. (You still pay this cost *if you actually need data from each of those files*, but 1. you might only need a tiny subset of the files, and 2. people expect doing big complicated calculations on gargantuan datasets to take a while anyway).
- Combining large, multifile datasets. When we open all these files as one dataset, there is still work to be done stitching all the files together into a single object - even if we don't bother to perform any validity checks. Virtualisation lets us do this *once*, and then cache the result. 

All in all - even on a posix filesystem, a virtualised dataset is a way better solution than a bunch of netCDF files, and unless we change from lustre, it's also workable from an inode perspective, unlike reserialising everything as zarr. (Again, I'm totally ignoring sharding and zipped zarr stores here.)

___
## Getting this all woring on Pawsey

Getting started on Acacia was really straightfoward. I've used GCS for some personal stuff before, thanks to their forever free tier, but I'd never played with S3/Ceph, and I'd only been on Setonix once or twice before. Luckily, their documentation has clearly been actually used, because I just followed along (No LLM helping me!) and was able to get set up in under an hour.

I decided to make 3 buckets on Pawsey: an `01deg` and `1deg` bucket under my user storage area, and a `ct-zarr-test` bucket under an ACCESS-NRI trial project storage area we had. 

Why under both? I'd heard off someone that Acacia might restrict policies (and cors headers) to only be freely available for buckets in user storage, and I wanted to test that. You only get 100GB of quota in your personal user storage, so if we wanted to virtualise real datasets, it would need to work for project storage too. 

One step at a time though...

## Virtualising Climate Data on Acacia 

I spent a few hours poking at the `01deg` bucket and trying to make a fully virtual, remotely-consumable dataset. The result: it works, and it's delightfully straightforward once you know the small gotchas.

What I tried

- Opening a single NetCDF from `s3://01deg` with `xarray` (anonymous reads, `h5netcdf`) to confirm the files and permissions.
- Using `virtualizarr` with `HDFParser` and an `S3Store` registry to build a virtual dataset across multiple files (`open_virtual_mfdataset`).
- Persisting the combined virtual dataset in several forms: an `icechunk` repository, a kerchunk JSON reference, and a parquet reference file for testing.

What happened

- `virtualizarr` successfully virtualised individual NetCDF files and combined them into a single `vds` that behaves like a normal `xarray` dataset.
- Some source files did not include explicit `ni`/`nj` coordinates; assigning `ni = np.arange(...)` and `nj = np.arange(...)` preserved spatial indexing through virtualisation.
- Writing kerchunk references (JSON or parquet) into the bucket made the dataset trivially consumable by `xarray.open_dataset` using the `reference://` Zarr backend and the appropriate remote options.
- Interactive plotting of small spatial slices was fast (order of 0.5–1s) when streaming via the kerchunk/virtual path. A very small `dask` client (single-thread workers) was enough for these tests.

Lessons & gotchas

- Anonymous reads: use `anon=True` with `client_kwargs` that point to `https://projects.pawsey.org.au` for public access. For write or private-read operations, provide `ACCESS_KEY_ID` / `SECRET_ACCESS_KEY` to `S3Store` and `icechunk`.
- Codec and backend quirks: register `numcodecs.zarr3` when required and prefer `consolidated=False` for these workflows; suppress the benign numcodecs warnings if they clutter logs. Note that suppressing warnings doesn't work if you're using a dask cluster, because suppressing warnings is a local operation and won't affect workers that are running in separate processes or machines. In that case, you may need to configure logging on the workers to ignore those specific warnings.
- S3 compatibility: set `force_path_style=True` and supply `endpoint_url` for S3-compatible backends like Acacia.

Why this matters

By keeping the data virtual and publishing kerchunk references, we can serve large model output without duplicating the heavy binary payloads. Consumers can stream exactly the slice they need, and the same kerchunk reference works for both anonymous viewers and credentialed clients (with the right storage options).

How to reproduce (quick)

1. Try an anonymous open: `xr.open_dataset("s3://01deg/output980/iceh.2146-01.nc", engine="h5netcdf", backend_kwargs={"storage_options": {"anon": True, "client_kwargs": {"endpoint_url": "https://projects.pawsey.org.au"}}})`
2. Build a combined virtual dataset with `open_virtual_mfdataset(..., parallel="dask", combine="nested", concat_dim="time")` and add `ni`/`nj` coords if missing.
3. Export `vds.vz.to_kerchunk(format='json')` and place the produced reference in the bucket. Open via `reference://` + `zarr` backend in `xarray` for remote plotting.

Next steps

- Add a small runnable example snippet to this page (anon vs credentialed), and a short checklist for writing kerchunk references back to Acacia.
- Test a larger time range with a small Dask cluster to validate performance at scale.

— end of notes

## Bucket permissions & CORS

To make the bucket readable anonymously and avoid unnecessary CORS preflight requests from browsers, apply two small changes to the bucket: a public read policy (for objects) and a conservative CORS configuration that only allows simple `GET`/`HEAD` requests from the origins you need.

1) Public read for objects

Create a `public-read.json` policy (replace `<BUCKET_NAME>` with your bucket, e.g. `01deg`):

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "PublicReadGetObject",
			"Effect": "Allow",
			"Principal": "*",
			"Action": "s3:GetObject",
			"Resource": "arn:aws:s3:::<BUCKET_NAME>/*"
		}
	]
}
```

Apply it with the AWS CLI (use `--endpoint-url` for Acacia / S3-compatible endpoints):

```bash
aws --endpoint-url=https://projects.pawsey.org.au s3api put-bucket-policy --bucket <BUCKET_NAME> --policy file://public-read.json
```

2) Minimal CORS (avoid preflight)

Browsers only send a CORS preflight (OPTIONS) when a request is not a "simple request" (method other than GET/HEAD/POST, or custom headers). To avoid preflights, ensure clients use simple GET/HEAD requests without custom headers, and configure a CORS policy that allows those methods from your origin(s). Example `cors.json` that permits simple GET/HEAD from any origin:

```json
{
	"CORSRules": [
		{
			"AllowedOrigins": ["*"],
			"AllowedMethods": ["GET", "HEAD"],
			"AllowedHeaders": ["*"] ,
			"MaxAgeSeconds": 3000
		}
	]
}
```

Apply it with:

```bash
aws --endpoint-url=https://projects.pawsey.org.au s3api put-bucket-cors --bucket <BUCKET_NAME> --cors-configuration file://cors.json
```

Notes and best practices

- Prefer restricting `AllowedOrigins` to the exact origin(s) that will fetch kerchunk references (instead of `*`) for better security.
- To truly avoid preflight requests, ensure clients do not send custom headers (for example, do not send `Authorization` or other non-simple headers) and use anonymous GET requests. If credentials are required, a CORS preflight is more likely and the server must accept it.
- For S3-compatible stores (Acacia), include `--endpoint-url` and `--profile` as needed; also use `force_path_style=True` in clients if required by your object store.
- Verify with:

```bash
aws --endpoint-url=https://projects.pawsey.org.au s3api get-bucket-policy --bucket <BUCKET_NAME>
aws --endpoint-url=https://projects.pawsey.org.au s3api get-bucket-cors --bucket <BUCKET_NAME>
curl -I https://projects.pawsey.org.au/<BUCKET_NAME>/reference.json
```

If you want, I can add the exact commands you used in your environment (I saw `aws --endpoint-url=https://projects.pawsey.org.au s3api put-bucket-policy --bucket icechunk-demo --policy file://public-read.json` in your terminal history) into the doc as a short reproducible snippet.

### Exact policy files and commands (from your terminal history)

Below are the actual policy JSON snippets and the exact AWS CLI commands pulled from your terminal history for the `icechunk-demo` bucket. Use these as copy-paste snippets.

`public-read.json` (actual content used):

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "PublicReadGetObject",
			"Effect": "Allow",
			"Principal": "*",
			"Action": "s3:GetObject",
			"Resource": "arn:aws:s3:::icechunk-demo/*"
		}
	]
}
```

`list-policy.json` (actual content used):

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Principal": {"AWS": ["arn:aws:iam:::user/cturner"]},
			"Action": "s3:ListBucket",
			"Resource": ["arn:aws:s3:::icechunk-demo"]
		}
	]
}
```

Exact CLI invocations (as found in your shell history):

```bash
aws --endpoint-url=https://projects.pawsey.org.au s3api put-bucket-policy --bucket icechunk-demo --policy file://public-read.json
aws --endpoint-url=https://projects.pawsey.org.au s3api put-bucket-policy --bucket icechunk-demo --policy file://list-policy.json
aws --endpoint-url=https://projects.pawsey.org.au s3api put-bucket-cors --bucket icechunk-demo --cors-configuration file://cors.json

# Variants with profile (seen in history):
aws --endpoint-url=https://projects.pawsey.org.au --profile=cturner s3api put-bucket-policy --bucket icechunk-demo --policy file://public-read.json
aws --endpoint-url=https://projects.pawsey.org.au --profile=cturner s3api put-bucket-cors --bucket icechunk-demo --cors-configuration file://cors.json

# Verification commands seen in history:
aws --endpoint-url=https://projects.pawsey.org.au s3api get-bucket-policy --bucket icechunk-demo
aws --endpoint-url=https://projects.pawsey.org.au s3api get-bucket-cors --bucket icechunk-demo
curl -I https://projects.pawsey.org.au/icechunk-demo/reference.json
```

These snippets are verbatim from your terminal selection and have been cleaned to valid JSON where needed. If you want me to keep the exact whitespace/quoting as in the raw history output I can paste that too, or I can write these files into the repo (`public-read.json`, `list-policy.json`, `cors.json`) for you.

___

# Trying to do the same thing on NCI/Nirin

## Nirin (NCI) — what made this harder

I reproduced the same virtualisation experiments on Nirin and hit a different set of operational limits compared with Acacia. Key points from the `virtualisation-tests-nirin.ipynb` run:

- Protocol & gateway differences: the Nirin setup used both a Swift URL and an S3 gateway (e.g. `https://cloud.nci.org.au:8080/swift/...` and `s3://ct-icechunk-root/...`). Kerchunk remote reads in the notebook used the Swift endpoint. Icechunk attempts used the S3 gateway with `endpoint_url` and credentials.
- No anonymous S3 reads: the S3 gateway on Nirin did not support anonymous access. This means consumers must either (a) provide credentials, (b) use signed URLs, or (c) have a trusted server in Nirin that fetches data on their behalf. Storing credentials in an interactive catalog or environment is insecure; the notebook notes running a small server in Nirin that can serve xarray reprs is a practical workaround.
- Performance variability (ARE vs remote): reading kerchunk references via Swift from an ARE session was much slower than remote reads (notably worse on the ARE interactive environment). There are likely multiple causes (Swift gateway performance, network path, authentication overhead); this made the Swift-based kerchunk workflow less usable from ARE.
- Icechunk vs Swift: icechunk currently lacks Swift support (noted in the notebook), so writing icechunk repositories into Nirin object storage required using the S3 gateway and credentials. That adds operational friction compared with Acacia where anonymous reads were straightforward.
- Credentials + client options: code in the notebook shows needing `access_key_id` / `secret_access_key`, `endpoint_url`, and `force_path_style=True` when constructing `S3Store` / `icechunk.s3_storage`. Remote options also used `asynchronous=True` and explicit `remote_options` for the `reference://` open.
- Container headers for Swift: the notebook includes CONTAINER_HEADERS (e.g. `X-Container-Read`, and CORS metadata) to allow anonymous Swift reads; depending on the object-store configuration you may need to set these metadata values on containers rather than using an S3 bucket policy.
- Compute-node access: a practical blocker is whether compute nodes can talk to the object storage endpoints (Nirin buckets) directly. If not, you must run a service inside Nirin or provide credentials on compute nodes — both have operational/security trade-offs.

Recommendations for working on Nirin

- Prefer kerchunk + Swift only if the Swift gateway provides acceptable performance for your interactive environment; otherwise prefer S3 gateway reads with signed URLs or a proxied service.
- If anonymous S3 reads are required, ask NCI support to enable them for the target bucket (or provide a suitable bucket policy); otherwise plan for a server-side proxy in Nirin that holds credentials and serves kerchunk/JSON to clients.
- For icechunk deployment, plan to use the S3 gateway with credentials and set `force_path_style=True` and the proper `endpoint_url` in clients; make sure any credentials are provisioned securely (not baked into catalog code).
- Benchmark both Swift and S3 remote reads from the actual interactive environment you expect to use (ARE, Gadi, etc.) — the notebook shows huge differences depending on where you run the client.
- Consider moving the interactive catalog (parquet/index files) into Nirin object storage if compute nodes can access it — that reduces cross-infra synchronization and simplifies access patterns.

Short summary: Nirin works, but the S3 gateway’s lack of anonymous access and the Swift/S3 performance differences add friction compared with Acacia. The practical workarounds are: enable anonymous access, use a small trusted proxy inside Nirin, or accept credentialed access with careful secret management.
