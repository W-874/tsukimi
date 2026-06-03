SPEC := $(if $(spec),$(spec),tsukimi.spec)

.PHONY: srpm
srpm:
	dnf -y install cargo git-core rpm-build rust zstd
	name="$$(rpmspec -q --srpm --qf '%{NAME}\n' "$(SPEC)" | head -n1)"; \
	version="$$(rpmspec -q --srpm --qf '%{VERSION}\n' "$(SPEC)" | head -n1)"; \
	workdir="$$(mktemp -d)"; \
	mkdir -p "$$workdir/.cargo"; \
	git archive --format=tar.gz --prefix="$$name-$$version/" -o "$(outdir)/$$name-$$version.tar.gz" HEAD; \
	cargo vendor --locked "$$workdir/vendor" > "$$workdir/.cargo/config.toml"; \
	tar --zstd -cf "$(outdir)/$$name-$$version-vendor.tar.zst" -C "$$workdir" .cargo vendor; \
	rpmbuild -bs "$(SPEC)" --define "_sourcedir $(outdir)" --define "_srcrpmdir $(outdir)"; \
	rm -rf "$$workdir"
