# 0x1 ZeroINK

> [Jackie](https://github.com/PyXMR2025)'s personal blog, built with [Hugo](https://gohugo.io/) + [PaperMod](https://github.com/adityatelange/hugo-PaperMod) and hosted on GitHub Pages and Cloudflare.

**URL**: [https://blog.zjackie.ink](https://blog.zjackie.ink)

[![License: Code MIT](https://img.shields.io/badge/License-Code%20MIT-blue.svg)](https://opensource.org/licenses/MIT)
[![License: Content CC BY-SA 4.0](https://img.shields.io/badge/License-Content%20CC%20BY--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-sa/4.0/)
[![Deploy Status](https://github.com/PyXMR2025/0x1-ZeroINK/actions/workflows/ci.yml/badge.svg)](https://github.com/PyXMR2025/0x1-ZeroINK/actions)

## Local Development

To run this blog locally, you need **Hugo (Extended edition)** installed.

1. **Clone the repository** (the theme is a Git submodule):
   ```bash
   git clone --recurse-submodules https://github.com/PyXMR2025/0x1-ZeroINK.git
   cd 0x1-ZeroINK
   ```
   If you already cloned without submodules:
   ```bash
   git submodule update --init --recursive
   ```

2. **Start the local preview server**:
   ```bash
   hugo server -D
   ```
   Open your browser and navigate to `http://localhost:1313`.

3. **Create a new post**:
   ```bash
   hugo new content content/posts/my-post.md
   ```
   Set `draft: false` in the front matter when you're ready to publish.

## Support & Donate

If you find this blog helpful or enjoy the content, consider supporting my work! Your contributions help keep the infrastructure running and fuel future open-source projects.

**Bitcoin (BTC)**:

```text
bc1pyhjz2ene2m88pwv8k4v3ql72c4pqqrvdvq8w75x2f0whzhf699aq2pl7yk
```

<details>
<summary><b>PGP Signature Verification</b> (Click to expand)</summary>

```text
-----BEGIN PGP SIGNED MESSAGE-----
Hash: SHA512

bc1pyhjz2ene2m88pwv8k4v3ql72c4pqqrvdvq8w75x2f0whzhf699aq2pl7yk
-----BEGIN PGP SIGNATURE-----

iHUEARYKAB0WIQQTplea0vauG7PFc++wm9vs48gNGQUCat9xvwAKCRCwm9vs48gN
GdoDAP9Thkjy0HjY+7PIta56ySt4PWXlrzirUlZgdQ8Z42LH8QEAuEIkVX94LiWY
SZY8BGnrKXhNVqjIpQHJ+4mXAYB7IQo=
=eDbG
-----END PGP SIGNATURE-----
```

</details>

**trocador.app**:

[bc1pyhjz2ene2m88pwv8k4v3ql72c4pqqrvdvq8w75x2f0whzhf699aq2pl7yk](https://trocador.app/anonpay/?ticker_to=btc&network_to=Mainnet&address=bc1pyhjz2ene2m88pwv8k4v3ql72c4pqqrvdvq8w75x2f0whzhf699aq2pl7yk&donation=True&simple_mode=False&amount=0.01&name=Jackie&description=%5B+Jackie+%5D+%211x0%21+%5B+Thanks+%5D&email=pyxmr2025@openenet.cn&bgcolor=00000000&direct=True)

## License

This project uses a **Dual License** approach to protect both the code and the content:

- **Code & Infrastructure**: The build scripts, configuration files, and custom code are licensed under the [MIT License](LICENSE).
- **Blog Content**: All original blog posts, articles, and media content (located in the `/content` directory) are licensed under the [Creative Commons Attribution-ShareAlike 4.0 International License (CC BY-SA 4.0)](https://creativecommons.org/licenses/by-sa/4.0/).

---

Made with ❤️ by [Jackie](https://github.com/PyXMR2025)
