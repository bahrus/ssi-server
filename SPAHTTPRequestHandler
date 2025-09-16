import os
from http.server import SimpleHTTPRequestHandler, HTTPServer
from urllib.parse import unquote


class SPAHTTPRequestHandler(SimpleHTTPRequestHandler):
    """
    A custom HTTP request handler that:
    - Supports Server-Side Includes (SSI): <!-- #include virtual="fragment.html" -->
    - Provides SPA fallback: serves index.html if a requested .html file is not found.
    """

    def send_head(self):
        """Override to add SPA fallback logic for missing .html files."""
        path = self.translate_path(self.path)
        f = None

        if os.path.isdir(path):
            # If directory, try index.html
            for index in ("index.html", "index.htm"):
                index_path = os.path.join(path, index)
                if os.path.exists(index_path):
                    path = index_path
                    break
            else:
                return super().send_head()

        if not os.path.exists(path):
            # SPA fallback: if request is for HTML and not found, serve root index.html
            if self.path.endswith(".html"):
                spa_fallback = os.path.join(os.getcwd(), "index.html")
                if os.path.exists(spa_fallback):
                    path = spa_fallback
                else:
                    self.send_error(404, "File not found")
                    return None
            else:
                return super().send_head()

        ctype = self.guess_type(path)
        try:
            f = open(path, 'rb')
        except OSError:
            self.send_error(404, "File not found")
            return None

        # Handle SSI if it's an HTML file
        if ctype == "text/html":
            content = f.read().decode("utf-8")
            f.close()
            content = self.handle_includes(content, os.path.dirname(path))
            encoded = content.encode("utf-8")
            self.send_response(200)
            self.send_header("Content-type", ctype)
            self.send_header("Content-Length", str(len(encoded)))
            self.end_headers()
            return self.BytesIO(encoded)

        # Otherwise serve normally
        self.send_response(200)
        self.send_header("Content-type", ctype)
        fs = os.fstat(f.fileno())
        self.send_header("Content-Length", str(fs.st_size))
        self.end_headers()
        return f

    def handle_includes(self, content, base_dir):
        """Process <!-- #include virtual="file" --> directives."""
        import re

        pattern = re.compile(r'<!--\s*#include\s+virtual="([^"]+)"\s*-->')

        def replace_include(match):
            include_path = match.group(1)
            include_path = os.path.join(base_dir, unquote(include_path))
            if os.path.exists(include_path) and os.path.isfile(include_path):
                try:
                    with open(include_path, "r", encoding="utf-8") as inc:
                        return inc.read()
                except Exception as e:
                    return f"<!-- Error including {include_path}: {e} -->"
            return f"<!-- File not found: {include_path} -->"

        return pattern.sub(replace_include, content)

    class BytesIOWrapper:
        """A minimal file-like wrapper around bytes for send_head return."""
        def __init__(self, data: bytes):
            import io
            self.buffer = io.BytesIO(data)

        def read(self, *args):
            return self.buffer.read(*args)

        def close(self):
            self.buffer.close()

    @staticmethod
    def BytesIO(data: bytes):
        return SPAHTTPRequestHandler.BytesIOWrapper(data)


def run(server_class=HTTPServer, handler_class=SPAHTTPRequestHandler, port=8000):
    server_address = ("", port)
    httpd = server_class(server_address, handler_class)
    print(f"Serving on port {port} (SPA + SSI enabled)...")
    httpd.serve_forever()


if __name__ == "__main__":
    run()