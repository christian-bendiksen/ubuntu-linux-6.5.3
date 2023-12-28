# Ubuntu 6.5.3 Kernel with sound fix

This custom Kernel is designed to solve the sound issues on the Lenovo Legion 7 Slim 16ARHA7, specifically addressing the problem of no sound. After installing this kernel, you should get sound back, though it might be at a lower volume than usual.

You can install this kernel using the .deb packages available in the "Release" section. Just use the command `sudo dpkg -i linux-*6.5.3*.deb` to get it set up.

Please be aware that installing this kernel is something you're doing at your own risk. I've put this together to help out, but I can't take responsibility for any issues that might come up as a result.



## Changes made to sound/pci/hda/cs35l41.c

Added functions to check for HID-specific configurations.

```c
static void common_configuration(struct cs35l41_hda *cs35l41,
				 struct device *physdev, int index)
{
	cs35l41->index = index;
	cs35l41->channel_index = 0;
	cs35l41->reset_gpio = gpiod_get_index(
		physdev, NULL, index, index ? GPIOD_OUT_LOW : GPIOD_OUT_HIGH);
	cs35l41->speaker_id = cs35l41_get_speaker_id(physdev, index, 0, 2);
}

// HID-specific configurations.
static int configure_for_hid(struct cs35l41_hda *cs35l41,
			     struct cs35l41_hw_cfg *hw_cfg,
			     struct device *physdev, int id, const char *hid)
{
	int index = (id == 0x40) ? 0 : 1;

	if (strncmp(hid, "CSC3551", 7) == 0) {
		common_configuration(cs35l41, physdev, index);
		hw_cfg->spk_pos = index;
		hw_cfg->gpio1.func = 1;
		hw_cfg->gpio1.valid = true;
		hw_cfg->gpio2.func = 0x02;
		hw_cfg->gpio2.valid = true;
		hw_cfg->bst_ipk = hw_cfg->bst_ind = hw_cfg->bst_cap = -1;
		hw_cfg->bst_type = (hw_cfg->bst_ind > 0 ||
				    hw_cfg->bst_cap > 0 ||
				    hw_cfg->bst_ipk > 0) ?
					   CS35L41_INT_BOOST :
					   CS35L41_EXT_BOOST;
		hw_cfg->valid = true;

		return 0;
	}

	if (strncmp(hid, "CLSA0100", 8) == 0) {
		hw_cfg->bst_type = CS35L41_EXT_BOOST_NO_VSPK_SWITCH;
	} else if (strncmp(hid, "CLSA0101", 8) == 0) {
		hw_cfg->bst_type = CS35L41_EXT_BOOST;
		hw_cfg->gpio1.func = CS35l41_VSPK_SWITCH;
		hw_cfg->gpio1.valid = true;
	} else {
		hw_cfg->valid = hw_cfg->gpio1.valid = hw_cfg->gpio2.valid =
			false;
		return -EINVAL;
	}

	return 1;
}
```

Refactored the cs35l41_no_acpi_dsd function.

```c
/*               
 * Device CLSA010(0/1) doesn't have _DSD so a gpiod_get by the label reset won't work.
 * And devices created by serial-multi-instantiate don't have their device struct
 * pointing to the correct fwnode, so acpi_dev must be used here.
 * And devm functions expect that the device requesting the resource has the correct
 * fwnode.       
 *               
 * also added support for CSC3551. However, sound is very low.
 */
static int cs35l41_no_acpi_dsd(struct cs35l41_hda *cs35l41,
			       struct device *physdev, int id, const char *hid)
{
	struct cs35l41_hw_cfg *hw_cfg = &cs35l41->hw_cfg;

	int res = configure_for_hid(cs35l41, hw_cfg, physdev, id, hid);
	if (res <= 0) {
		if (res == -EINVAL) {
			dev_err(cs35l41->dev,
				"Error: ACPI _DSD Properties are missing for HID %s.\n",
				hid);
		}
		return res;
	}
	/* check I2C address to assign the index */
	int index = (id == 0x40) ? 0 : 1;
	common_configuration(cs35l41, physdev, index);
	hw_cfg->spk_pos = cs35l41->index;
	hw_cfg->gpio2.func = CS35L41_INTERRUPT;
	hw_cfg->gpio2.valid = true;
	hw_cfg->valid = true;

	return 0;
}
```

Thanks to https://bugzilla.kernel.org/show_bug.cgi?id=216194 for all the necessary information.
