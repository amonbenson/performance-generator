from SCons.Script import *
from SCons.Node.FS import File
from SCons.Errors import UserError

import os
import io
import pypdf
import pypdf.generic
import pypdf.annotations
import fpdf
import itertools

def pdf_move_keep_annotations(target: File, source: File, env: Environment):
    # if the target does not exist, copy the source directly
    if not target.exists():
        env.Execute(Move(target, source))
        return

    # move the previous target file out of the way to a backup name
    previous: File = env.ReplaceExt(target, ".pdf", " Backup.pdf")
    env.Execute(Move(previous, target))

    # open both source and target
    with open(str(source), "rb") as fs:
        with open(str(previous), "rb") as ft:
            source_doc = pypdf.PdfReader(fs)
            previous_doc = pypdf.PdfReader(ft)

            # copy the annotations from the previous target to the source pdf
            if len(source_doc.pages) != len(previous_doc.pages):
                raise UserError("Cannot transfer annotations: Number of pages changed.")

            for source_page, previous_page in zip(source_doc.pages, previous_doc.pages):
                if "/Annots" in previous_page:
                    source_page[pypdf.generic.NameObject("/Annots")] = previous_page["/Annots"]

            # write the target file
            writer = pypdf.PdfWriter()
            writer.append(source_doc)

            with open(str(target), "wb") as f:
                writer.write(f)

    # delete the source
    env.Execute(Delete(source))

def pdf_merge_action(target: list[File], source: list[File], env: Environment):
    watermarks = env.get("WATERMARKS", itertools.repeat(None))
    merger = pypdf.PdfWriter()

    # append all input files
    for s, watermark in zip(source, watermarks):
        starting_page = merger.get_num_pages()

        if isinstance(s, File):
            # read from file
            with open(str(s), "rb") as f:
                reader = pypdf.PdfReader(f)
                merger.append(reader)
        else:
            # create blank page
            merger.add_blank_page()

        # add the watermarks
        if watermark and False: # disable watermarking for now
            # get the target page that the watermark should be applied to
            page = merger.get_page(starting_page)

            # create an overlay page to draw onto
            unit = "mm"
            k = fpdf.get_scale_factor(unit)
            format = (page.mediabox[2] / k, page.mediabox[3] / k)
            overlay = fpdf.FPDF(format=format, unit=unit)
            overlay.add_page()

            if "nr" in watermark:
                overlay.set_font("Arial", style="B", size=24)
                overlay.text(50, 150, "#" + str(watermark["nr"]))

            if "title" in watermark:
                overlay.set_font("Arial", style="B", size=24)
                overlay.text(50, 250, str(watermark["title"]))

            # apply the overlay using pypdf
            overlay_pypdf = pypdf.PdfReader(io.BytesIO(overlay.output())).pages[0]
            merger.pages[starting_page].merge_page(page2=overlay_pypdf)

    # write the target file
    with open(str(target[0]), "wb") as f:
        merger.write(f)

    merger.close()

def docx_to_pdf_action(target: list[File], source: list[File], env: Environment):
    # run the converter
    tmp_target: File = env.ReplaceExt(target[0], ".pdf", ".tmp.pdf")
    env.Execute(f"docx2pdf \"{source[0].get_abspath()}\" \"{tmp_target.get_abspath()}\"")

    # replace the target while keeping existing annotations
    pdf_move_keep_annotations(target[0], tmp_target, env)

# setup environment and custom builders/methods
env = Environment()
env.Append(
    ENV = {
        "PATH": os.environ["PATH"],
    },
    BUILDERS = {
        "Docx2Pdf": Builder(action=docx_to_pdf_action, suffix=".pdf", src_suffix=".docx", single_source=True),
        "PdfMerge": Builder(action=Action(pdf_merge_action, varlist=["WATERMARKS"]), suffix=".pdf", src_suffix=".pdf"),
    }
)

def replace_ext(env, f, old_ext, new_ext):
    return env.File(str(f).rsplit(old_ext, 1)[0] + new_ext)
env.AddMethod(replace_ext, "ReplaceExt")

# include all script files
SConscript("Songs/SConscript", exports="env")
SConscript("Performances/SConscript", exports="env")

# specify default alias targets
env.Default("songs", "performances")
