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
## Getting this all working on Pawsey

Getting started on Acacia was really straightfoward. I've used GCS for some personal stuff before, thanks to their forever free tier, but I'd never played with S3/Ceph, and I'd only been on Setonix once or twice before. Luckily, their documentation has clearly been actually used, because I just followed along (No LLM helping me!) and was able to get set up in under an hour.

I decided to make 3 buckets on Pawsey: an `01deg` and `1deg` bucket under my user storage area, and a `ct-zarr-test` bucket under an ACCESS-NRI trial project storage area we had. 

Why under both? I'd heard off someone that Acacia might restrict policies (and cors headers) to only be freely available for buckets in user storage, and I wanted to test that. You only get 100GB of quota in your personal user storage, so if we wanted to virtualise real datasets, it would need to work for project storage too. 

One step at a time though...

## Virtualising Climate Data on Acacia 

After I got the bucket setup and some data copied in, I spent a few hours poking at the `01deg` bucket and trying to make a virtual, remotely-consumable dataset. A few hours later, I had exaclt what I was after. Here's how it went down: see [this notebook](https://github.com/charles-turner-1/PawseyVirtualisationTests/blob/main/pawsey/vz_01deg.ipynb) if you want to see the full process.

What I tried

- Open a single NetCDF over S3 to check permissions and basic reads: `xarray` + `h5netcdf` with anonymous client options against `s3://01deg`.
- Use `virtualizarr` (the `HDFParser` + `S3Store` combo) to build a virtual dataset across many files via `open_virtual_mfdataset`.
- Persist the combined virtual dataset in a few formats for downstream testing: an `icechunk` repo, a kerchunk JSON reference, and a parquet reference file.

What actually happened

- `virtualizarr` did exactly what it promised: the individual netCDFs got virtualised and stitched into one `vds` that behaves just like a normal `xarray` dataset.
- A few source files lacked explicit `ni`/`nj` coords. I assigned `ni = np.arange(...)` and `nj = np.arange(...)` before virtualising and that preserved spatial indexing through the pipeline.
- Dropping kerchunk references (JSON/parquet) into the bucket made the dataset trivial to open with `xarray.open_dataset("reference://...", engine="zarr", backend_kwargs={...})`.
- Small interactive plots (tiny spatial slices) were snappy — roughly 0.5–1s — when streamed via the kerchunk/virtual path. A tiny local `dask` client (single-thread workers) was sufficient for these quick interactions.

Lessons and gotchas

- Anonymous reads: for public access use `anon=True` and `client_kwargs` pointing at `https://projects.pawsey.org.au`. If you need writes or private reads then supply `ACCESS_KEY_ID` / `SECRET_ACCESS_KEY` to `S3Store` and `icechunk`.
- Backend oddities: register `numcodecs.zarr3` when needed and prefer `consolidated=False` for this workflow. If warnings get noisy, suppress them locally — but remember that suppression doesn't propagate to separate Dask workers, so you may need to configure worker logging.
- S3 quirks: use `force_path_style=True` and set the right `endpoint_url` for Acacia (or any S3-compatible store).

Why this matters

Publishing kerchunk references lets you serve arbitrarily large model output without copying the petabytes. Consumers stream exactly what they need and the same reference file works for both anonymous and credentialed clients (provided the storage options are set correctly).

Quick reproduction

1. Anonymous open (sanity check):

```bash
python -c "import xarray as xr; xr.open_dataset('s3://01deg/output980/iceh.2146-01.nc', engine='h5netcdf', backend_kwargs={'storage_options': {'anon': True, 'client_kwargs': {'endpoint_url': 'https://projects.pawsey.org.au'}}})"
```

2. Build a combined virtual dataset:

Use `open_virtual_mfdataset(..., parallel='dask', combine='nested', concat_dim='time')` and add missing `ni`/`nj` coords before virtualising.

3. Export kerchunk reference and open remotely:

```py
# inside notebook
vds.vz.to_kerchunk(format='json')
# upload the produced reference.json to your bucket and then open with reference:// + zarr backend
```

Next things to do

- Add a compact runnable snippet to this page showing anonymous vs credentialed opens.
- Spin up a tiny Dask cluster and test a longer time slice to validate behaviour at scale.

— end of notes

## Bucket permissions & CORS

If you want anonymous, fast reads from browsers you need just two small changes on the bucket: a public-read policy for objects and a minimal CORS configuration that only allows simple `GET`/`HEAD` calls from the origins you actually use. That keeps preflight traffic low and avoids forcing users through auth flows.

1) Public read for objects

Create a `public-read.json` (replace `<BUCKET_NAME>`):

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

Apply it using the AWS CLI (remember `--endpoint-url` for Acacia):

```bash
aws --endpoint-url=https://projects.pawsey.org.au s3api put-bucket-policy --bucket <BUCKET_NAME> --policy file://public-read.json
```

2) Minimal CORS (avoid preflight)

Browsers send OPTIONS preflight when a request isn't "simple" (e.g. custom headers). To avoid that, make your clients issue plain GET/HEAD without custom headers and set a CORS policy that allows those methods. A permissive example (but prefer locking down origins):

```json
{
  "CORSRules": [
    {
      "AllowedOrigins": ["*"],
      "AllowedMethods": ["GET", "HEAD"],
      "AllowedHeaders": ["*"],
      "MaxAgeSeconds": 3000
    }
  ]
}
```

Apply with:

```bash
aws --endpoint-url=https://projects.pawsey.org.au s3api put-bucket-cors --bucket <BUCKET_NAME> --cors-configuration file://cors.json
```

Notes and best practice

- Prefer restricting `AllowedOrigins` to the exact origin(s) rather than `*` for security.
- To really avoid preflight, clients must not send custom headers (no `Authorization`). If you must use credentials, expect preflights and configure the server accordingly.
- For Acacia and other S3-compatible stores: always pass `--endpoint-url` and consider `force_path_style=True` in clients.
- Quick checks:

```bash
aws --endpoint-url=https://projects.pawsey.org.au s3api get-bucket-policy --bucket <BUCKET_NAME>
aws --endpoint-url=https://projects.pawsey.org.au s3api get-bucket-cors --bucket <BUCKET_NAME>
curl -I https://projects.pawsey.org.au/<BUCKET_NAME>/reference.json
```

If you want I can drop the exact CLI invocations you used into this repo (public-read.json, list-policy.json, cors.json) so guys can copy-paste.

### Exact policy files & commands (from your shell history)

These are the snippets you used for `icechunk-demo` (cleaned up):

`public-read.json`:

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

`list-policy.json`:

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

Commands you ran (cleaned):

```bash
aws --endpoint-url=https://projects.pawsey.org.au s3api put-bucket-policy --bucket icechunk-demo --policy file://public-read.json
aws --endpoint-url=https://projects.pawsey.org.au s3api put-bucket-policy --bucket icechunk-demo --policy file://list-policy.json
aws --endpoint-url=https://projects.pawsey.org.au s3api put-bucket-cors --bucket icechunk-demo --cors-configuration file://cors.json

# variants with profile
aws --endpoint-url=https://projects.pawsey.org.au --profile=cturner s3api put-bucket-policy --bucket icechunk-demo --policy file://public-read.json
aws --endpoint-url=https://projects.pawsey.org.au --profile=cturner s3api put-bucket-cors --bucket icechunk-demo --cors-configuration file://cors.json

# verify
aws --endpoint-url=https://projects.pawsey.org.au s3api get-bucket-policy --bucket icechunk-demo
aws --endpoint-url=https://projects.pawsey.org.au s3api get-bucket-cors --bucket icechunk-demo
curl -I https://projects.pawsey.org.au/icechunk-demo/reference.json
```

These are the cleaned, copy-paste ready versions. If you prefer the raw history output I can paste that too, or I can add these files to the repo for easy reuse.

___

# Okay... why?

The guys over at carbonplan have been working on [this visualisation tool](https://github.com/carbonplan/zarr-layer), which I came across recently. I figured this was exactly what we needed in order to produce the over the wire data exploration tool which people wanted.

Using zarr-layer, [zarrita](https://zarrita.dev/) (a typescript library for dealing with zarr files), and Claude Sonnet 4.6 to speed up the process, I managed to generate [this interactive data explorer](https://charles-turner-1.github.io/personal-homepage/#/projects/zarr-data-streamer) in about 2 hours.

This demo is still obviously very rough and ready - it's still on native grids for a start - but it's clearly a massive improvement over current data exploration tools, and surprisingly easy to get working. It turns out you can get a lot done by grabbing onto the coattails of very smart people.

___


# Trying the same thing on NCI / Nirin

So virtualising and streaming data from Acacia works remarkably well. Great!

Unfortunately, we deal with making climate data available to scientists in Austalia, and most of Australia's trove of climate data is hosted on Gadi. This means we have two options (ignoring the sit on our hands approach):
1. Move the data to Acacia and host it from there. This is a bit of a non starter for a lot of reasons - but most of the work the climate science community does on HPC is on Gadi. So we'd either have to duplicate the data, or stream it back to Gadi. Neither of these are really tenable.
2. See if we can do the same thing on NCI's object storage, Nirin.

## Why Nirin felt harder

I ran the same experiments on Nirin and hit a different set of operational annoyances compared with Acacia. Short version:

- Protocol & gateway differences: Nirin exposes both Swift and an S3 gateway (e.g. `https://cloud.nci.org.au:8080/swift/...` and `s3://ct-icechunk-root/...`). In these experiments, I accessed the kerchunk reference remotely  via the Swift endpoint while icechunk tests used the S3 gateway + credentials. This is because apparently public reads on Nirin buckets can only be achieved via Swift, not S3.
- No anonymous S3 reads: the S3 gateway did not allow anonymous access. That means consumers need credentials, signed URLs, or a trusted proxy inside Nirin to serve data. This means that if we want to read the icechunk store remotely, we need to provide credentials, as icechunk doesn't support swift.
- Performance variability: reads from ARE (interactive environment) over Swift were noticeably slower than remote reads. I think there is some throttling of downloads into an ARE session, for reasons I don't totally understand.
 With that said, it is still substantially faster to read the virtual references than the raw data: 2 minutes 40 for the non virtualised dataset, 45s for the kerchunk reference over swift into an ARE session, and 2.6 seconds for the kerchunk reference remotely.
- Weirdly, you don't see the same performance issues with icechunk: it reads in 1.6 seconds from within an ARE session, but takes 2.2 remotely. This makes a bit more sense *a priori*, but it's weird that we see such a massive slowdown via Swift, when it's actually faster via s3. 
- Icechunk vs Swift: icechunk doesn't speak Swift, so writing icechunk repos required the S3 gateway and credentials — more friction than the Acacia case. 

  This might be less of a big deal than it seems. You can't stream zarr from an icechunk store straight into the browser, so if you want to use the same set of virtual references for both remote data streaming/exploration, and to speed up reads on Gadi, you need to use kerchunk (or set up a server to handle getting data from icechunk to the browser). 

  However, it does mean that the slowdown of kerchunk over Swift is more of an issue. Either we'd need to figure out how to speed up access to the kerchunk reference over Swift, or make the s3 gateway support anonymous reads and then read the kerchunk reference via that.
- Compute-node access: the other big operational question for putting virtual reference datasets on Gadi is whether compute nodes can talk to Nirin endpoints directly. If they can't, then you'd need to have these references on the posix filesystem somewhere too. This is fine - in theory - but the number of these we'd be creating increases the risk that something gets out of sync. It seems like the atomicity guarantees that Swift provides aren't particularly great either...
- Documentation: The documentation for Acacia is really pretty clear, and easy to follow. Acacia is built on Ceph, and access if using the aws cli. In contrast, Nirin has multiple access methods (Swift, S3 gateway) and the documentation is a quite a bit less complete. It also doesn't seem to be up to date: lots of things I tried seemed to no longer be true. It also contains gems like this:

    > The Nirin Cloud integrated object storage service supports a subset of the Swift and S3 access control mechanisms, with additional constraints which make them very limited for practical use. The NCI Cloud Team does not recommend the use of anything beyond the simplest case of enabling public read access to a container, which can be done via a simple check box for each container in the dashboard's Object Store → Containers tab.

  Which subset does the Nirin object stoage support? Not documented. And even though it explicitly states that you can set public read access, this only seems to apply to using the Swift Access protocol. In fact, it seems that the bucket policies are not set on the bucket itself, but on the access protocol. So you can have a totally different set of policies depending on whether you try to access it via Swift, or via s3api.

  If someone at NCI knows how to fix this, please let me know!
    

Recommendations for Nirin

- Use kerchunk + Swift only if the Swift gateway performs well from the interactive environment you plan to use; otherwise use S3 reads with signed URLs or a proxy.
- If anonymous S3 reads are required, request that from NCI support or plan for a proxy service in Nirin that holds credentials and serves kerchunk JSON.
- For icechunk, plan to use the S3 gateway with credentials and `force_path_style=True`; handle secrets securely (don’t bake them into notebooks 🫠).
- Benchmark both Swift and S3 reads from your intended client (ARE, Gadi interactive node, etc.) — performance varies a lot depending on where you run the client.
- Consider placing the interactive catalog/parquet files in Nirin object storage (if compute nodes can read them) to reduce cross-site syncing.

Bottom line: Nirin is usable, but the lack of anonymous S3 reads and the Swift/S3 performance differences make it more operationally fiddly than Acacia. The usual workarounds are: enable anonymous access, run a small trusted proxy inside Nirin, or accept credentialed access with careful secret management.
